---
type: post
title: GoでSSHサーバにラインエディタが欲しいなら golang.org/x/crypto/ssh/terminal
date: 2016-12-01
hero: "/line-editor-in-go-ja/rafaela-biazi-2N1lJCLrCj0-unsplash.jpg"
tags:
  - SSH
  - Go
authors:
  - Yutaka Ichibangase
---
# はじめに

GoはSSHサーバを書くのもかんたんです。ほとんどの場合、あなたのSSHサーバはユーザからのコマンド入力を受け付けるものでしょう。その場合、キー入力の列を文字列に変換するラインエディタと呼ばれるものが必要になります。Goでは`golang.org/x/crypto/ssh/terminal`がそれです。

# SSHサーバを書く

SSHサーバを書きましょう。GoでSSHサーバを書くには`golang.org/x/crypto/ssh`を使います。

ここでは例として入力行を送り返すだけの単純なSSHサーバを考えます。120行ほどあるので、読み飛ばしてもらってかまいません。重要なポイントは以下の3点だけです。

- プロンプトを表示する `w.WriteString(prompt)`
- ユーザの入力行を受け取る `l, _, err := r.ReadLine()`
- 入力行を送り返す `w.WriteString("\r\nYou've typed: " + string(l) + "\n")`

```go
package main

import (
	"bufio"
	"fmt"
	"io/ioutil"
	"log"
	"net"
	"os/exec"

	"golang.org/x/crypto/ssh"
)

func main() {
	key, err := privateKey()
	if err != nil {
		log.Fatalf("failed to load private key: %v", err)
	}

	config := &ssh.ServerConfig{NoClientAuth: true}
	config.AddHostKey(key)

	listener, err := net.Listen("tcp", "0.0.0.0:2022")
	if err != nil {
		log.Fatalf("failed to listen on 2022: %v", err)
	}

	for {
		tcp, err := listener.Accept()
		if err != nil {
			log.Printf("failed to accept tcp connection: %v", err)
			continue
		}

		_, chans, reqs, err := ssh.NewServerConn(tcp, config)
		if err != nil {
			log.Printf("failed to handshake: %v", err)
			continue
		}

		go ssh.DiscardRequests(reqs)
		go handleChannels(chans)
	}
}

func handleChannels(chans <-chan ssh.NewChannel) {
	for c := range chans {
		go handleChannel(c)
	}
}

func handleChannel(c ssh.NewChannel) {
	if t := c.ChannelType(); t != "session" {
		msg := fmt.Sprintf("unknown channel type: %s", t)
		c.Reject(ssh.UnknownChannelType, msg)
		return
	}

	conn, _, err := c.Accept()
	if err != nil {
		log.Printf("failed to accept channel: %v", err)
		return
	}
	defer conn.Close()

	r := bufio.NewReader(conn)
	w := bufio.NewWriter(conn)
	prompt := "> "

	for {
		if _, err := w.WriteString(prompt); err != nil {
			log.Printf("failed to write: %v", err)
			return
		}

		if err := w.Flush(); err != nil {
			log.Printf("failed to flush: %v", err)
			return
		}

		l, _, err := r.ReadLine()
		if err != nil {
			log.Printf("failed to read: %v", err)
			return
		}

		if _, err := w.WriteString("\r\nYou've typed: " + string(l) + "\n"); err != nil {
			log.Printf("failed to write: %v", err)
			return
		}

		if err := w.Flush(); err != nil {
			log.Printf("failed to flush: %v", err)
			return
		}
	}
}

func privateKey() (ssh.Signer, error) {
	b, err := privateKeyBytes()
	if err != nil {
		return nil, err
	}

	return ssh.ParsePrivateKey(b)
}

func privateKeyBytes() ([]byte, error) {
	if key, err := ioutil.ReadFile("example.rsa"); err == nil {
		return key, err
	}

	if err := exec.Command("ssh-keygen", "-f", "example.rsa", "-t", "rsa", "-N", "").Run(); err != nil {
		return nil, err
	}

	return ioutil.ReadFile("example.rsa")
}
```

このSSHサーバを試してみましょう。`go run [ファイル名]`でサーバを起動し、別ターミナルで`ssh -p2022 localhost`でクライアントを起動します。すぐにこれがダメであることがわかるでしょう。

- 入力途中の状態が表示されない
- `Ctrl-d`で終了しない
- リターンキーが反応しない

つまり、ぜんぜんダメです。これらはすべて、ラインエディタが使われていないことに起因します。

# SSHサーバにラインエディタを足す

ラインエディタは編集状態を画面に表示しつつユーザからのキー入力を受け取り、ユーザがリターンキーを打つと入力行を確定してプログラムに渡すという処理をするライブラリで、代表的なCでの実装にreadline、libedit、linenoiseなどがあります。

上述のSSHサーバで入力途中の状態が表示されないのは、キー入力の度に画面に表示する内容を更新していないからです。`Ctrl-d`が効かないのは、そのキーをSSHサーバが解釈する機能をまだ持っていないからです。そして、SSHサーバが受け取っているのは改行コード（LF `0x0A`またはCR+LF `0x0D, 0x0A`）ではなく、リターンキー（CR `0x0D`）であるため、`r.ReadLine()`では入力の確定を検知できないのです。すなわち、SSHサーバが受け取っているのはキー入力の列であり、文字列ではない、といえるでしょう。ラインエディタはキー入力の列から行単位の文字列を取り出すことができます。

SSHサーバにラインエディタを足しましょう。`golang.org/x/crypto/ssh/terminal`の出番です。これも100行あるので読み飛ばしてもらってかまいません。ポイントは以下の3点です。

- ラインエディタ`Terminal`を作成する `t := terminal.NewTerminal(conn, "> ")`
- ユーザ入力行を受け取る `l, err := t.ReadLine()`
- 入力行を送り返す `t.Write([]byte("You've typed: " + string(l) + "\r\n"))`

プロンプトの表示は`Terminal`がやってくれます。

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net"
	"os/exec"

	"golang.org/x/crypto/ssh"
	"golang.org/x/crypto/ssh/terminal"
)

func main() {
	key, err := privateKey()
	if err != nil {
		log.Fatalf("failed to load private key: %v", err)
	}

	config := &ssh.ServerConfig{NoClientAuth: true}
	config.AddHostKey(key)

	listener, err := net.Listen("tcp", "0.0.0.0:2022")
	if err != nil {
		log.Fatalf("failed to listen on 2022: %v", err)
	}

	for {
		tcp, err := listener.Accept()
		if err != nil {
			log.Printf("failed to accept tcp connection: %v", err)
			continue
		}

		_, chans, reqs, err := ssh.NewServerConn(tcp, config)
		if err != nil {
			log.Printf("failed to handshake: %v", err)
			continue
		}

		go ssh.DiscardRequests(reqs)
		go handleChannels(chans)
	}
}

func handleChannels(chans <-chan ssh.NewChannel) {
	for c := range chans {
		go handleChannel(c)
	}
}

func handleChannel(c ssh.NewChannel) {
	if t := c.ChannelType(); t != "session" {
		msg := fmt.Sprintf("unknown channel type: %s", t)
		c.Reject(ssh.UnknownChannelType, msg)
		return
	}

	conn, _, err := c.Accept()
	if err != nil {
		log.Printf("failed to accept channel: %v", err)
		return
	}
	defer conn.Close()

	t := terminal.NewTerminal(conn, "> ")

	for {
		l, err := t.ReadLine()
		if err != nil {
			log.Printf("failed to read: %v", err)
			return
		}

		if _, err := t.Write([]byte("You've typed: " + string(l) + "\r\n")); err != nil {
			log.Printf("failed to write: %v", err)
			return
		}
	}
}

func privateKey() (ssh.Signer, error) {
	b, err := privateKeyBytes()
	if err != nil {
		return nil, err
	}

	return ssh.ParsePrivateKey(b)
}

func privateKeyBytes() ([]byte, error) {
	if key, err := ioutil.ReadFile("example.rsa"); err == nil {
		return key, err
	}

	if err := exec.Command("ssh-keygen", "-f", "example.rsa", "-t", "rsa", "-N", "").Run(); err != nil {
		return nil, err
	}

	return ioutil.ReadFile("example.rsa")
}
```

ラインエディタを導入したことにより、入力途中の状態が逐次画面に反映され、`Ctrl-d`で終了することも出来ます。

# サクセス

![balloon.png](/images/balloon.png)

["Gopher Stickers"](https://github.com/tenntenn/gopher-stickers) by [Takuya Ueda](https://twitter.com/tenntenn) is licensed under [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/deed.ja)

# おわりに

GoでSSHサーバを書く際に、`golang.org/x/crypto/ssh/terminal`でラインエディタをかんたんに追加できることを紹介しました。

この記事は [Go (その2) Advent Calendar 2016](http://qiita.com/advent-calendar/2016/go2) の１日目の記事として書かれました。そこでは「Goをはじめる際の注意点について書きます」と予告していました。

この記事はGo初心者である僕が、GoにはSSHサーバのライブラリはあるが、併用するラインエディタが不足していると早とちりして[自前で書いてしまった](https://github.com/ichiban/linesqueak)という失敗に由来します。

Goをはじめる際の注意点です。あなたが思いついた便利なライブラリはだいたい `golang.org/x/` の下にあります。

