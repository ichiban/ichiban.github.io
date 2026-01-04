---
type: post
title: Goの実行ファイルにZIPでリソースを埋め込む
date: 2018-12-01
hero: "/zip-asset-embedding-in-go/tomas-sobek-nVqNmnAWz3A-unsplash.jpg"
tags:
  - Go
  - zip
  - template
authors:
  - Yutaka Ichibangase
---
# はじめに

ZIPを用いた実行ファイルへのリソースの埋め込み方法があることを紹介し、実際にGoの `archive/zip` と `zip` コマンドと `cat` コマンド（と確認のために `unzip` コマンド）を用いたリソース埋め込みの例を解説する。

# Goのリソース埋め込み

Goでアプリケーションを書く際、CSSやJavaScript、画像、テンプレートといったリソースは実行ファイルとは別に配置するか実行ファイルの中に埋め込むかしないといけない。リソースの埋め込みにはコードジェネレータを用いる方法とZIPを使う方法がある。

## コードジェネレータ

リソースの埋め込みでよく紹介されるのはコードジェネレータを用いてリソースをGoのソースコードに変換するという方法で、実際 [Awesome Go](https://awesome-go.com/) の [Resource Embeddingの項目](https://awesome-go.com/#resource-embedding) にあるのはコードジェネレータを用いてGoのソースコードに変換するアプローチのものしかない。

## ZIP

一方、リソースをZIPファイルにまとめて実行ファイルに追記するという埋め込み方もある。既に [Zgok](https://github.com/srtkkou/zgok) というライブラリがあり、作者の @srtkkou さんが解説（[Golangで静的ファイルをバイナリに含めるライブラリを書いてみた](https://qiita.com/srtkkou/items/131b251a5fa29213ac5e)）を書かれている。

[ZIPファイルの仕様](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)ではself-extractingなZIPファイルはターゲットプラットフォームごとの展開コードを含まなければならないとある。

>  4.1.9 ZIP files MAY be streamed, split into segments (on fixed or on
>   removable media) or "self-extracting".  Self-extracting ZIP 
>   files MUST include extraction code for a target platform within 
>   the ZIP file. 

また、[`zip` コマンドのmanページ](https://linux.die.net/man/1/zip)にはself-extractingな実行可能ファイル兼アーカイブは既存のアーカイブにSFXスタブを前置することで作られるとある。

> A self-extracting executable archive is created by prepending the SFX stub to an existing archive.

これらのことから、実行ファイルそれ自体をZIPファイルとして取り扱うGoのコマンドを書き、その実行ファイルの末尾にZIPファイルを足せばリソースを埋め込めることがわかる。[^wtf-is-sfx-stub]

# ZIPでのリソース埋め込みの例

ここでは `main.go` と `assets/templates/hello.tmpl` だけからなるシンプルな例について考える。

```
$ tree .
.
├── assets
│   └── templates
│       └── hello.tmpl
└── main.go

2 directories, 2 files
```

`asserts/templates.hello.tmpl` は以下に示すような単純なテンプレートである。

```
Hello, {{.}}!

```

また、 `main.go` は自身の実行ファイル `os.Executable()` をZIPファイルとして読み出し、一時ディレクトリに展開する。その後、展開されたリソース中のテンプレートを用いて `Hello, World!` を出力する。

`archive/zip` を使うときの注意点として、アーカイブ中のパスに `..` が無いことをチェックする必要がある。 `..` を許容すると[任意のコマンドを実行される脆弱性](https://snyk.io/research/zip-slip-vulnerability)につながる。

```go
package main

import (
	"archive/zip"
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"os"
	"path/filepath"
	"strings"
	"text/template"
)

func main() {
	exec, err := os.Executable()
	if err != nil {
		log.Fatalf("failed to get executable: %v", err)
	}

	r, err := zip.OpenReader(exec)
	if err != nil {
		log.Fatalf("failed to get zip reader: %v", err)
	}

	dir, err := ioutil.TempDir("", filepath.Base(exec))
	if err != nil {
		log.Fatalf("failed to open template directory: %v", err)
	}
	defer os.RemoveAll(dir)

	for _, f := range r.File {
		if err := extract(f, dir); err != nil {
			log.Fatalf("failed to extract file: %v", err)
		}
	}

	log.Printf("assets: %s", dir)

	t := template.Must(template.ParseGlob(filepath.Join(dir, "templates", "*")))
	if err := t.ExecuteTemplate(os.Stdout, "hello.tmpl", "World"); err != nil {
		log.Fatalf("t.ExecuteTemplate() failed: %v", err)
	}
}

func extract(f *zip.File, dir string) error {
	if strings.Contains(f.Name, "..") {
		// Zip Slip!
		return fmt.Errorf("file path '%s' contains '..'", f.Name)
	}

	path := filepath.Join(dir, filepath.Clean(f.Name))

	if f.Mode().IsDir() {
		if err := os.MkdirAll(path, f.Mode()); err != nil {
			return err
		}
		return nil
	}

	r, err := f.Open()
	if err != nil {
		return err
	}

	tf, err := os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, f.Mode())
	if err != nil {
		return err
	}
	defer tf.Close()

	_, err = io.Copy(tf, r)
	return err
}
```

## 実行ファイルの作成

まず通常の方法で実行ファイルを作成する。

```
$ go build -o hello
```

これにはリソースが埋め込まれていないため、必ず失敗する。

```
$ ./hello 
2018/11/25 17:15:32 failed to get zip reader: zip: not a valid zip file
```

## ZIPファイルの作成

次にリソースをまとめたZIPファイルを作成する。 `zip` コマンドは実行時のカレントディレクトリを基準にするため `assets` ディレクトリの中で実行する必要がある。

```
$ cd assets
$ zip -r ../assets.zip .
  adding: templates/ (stored 0%)
  adding: templates/hello.tmpl (stored 0%)
$ cd ..
```

正常に作られたリソースのZIPファイルにはパスに `..` や `assets` が含まれない。

```
$ unzip -l assets.zip
Archive:  assets.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  11-25-2018 16:13   templates/
       14  11-05-2018 21:31   templates/hello.tmpl
---------                     -------
       14                     2 files
```

## 実行ファイル兼ZIPファイルの作成

次に、実行ファイルとZIPファイルを連結する。

```
$ cat hello assets.zip > hello-bundled
$ chmod +x hello-bundled
```

単純に連結しただけだとZIPファイル中のオフセット値が実行ファイルのサイズ分ずれてしまっているのでまだ失敗する。

```
$ ./hello-bundled
2018/11/25 17:16:37 failed to get zip reader: zip: not a valid zip file
```

```
$ unzip -l hello-bundled 
Archive:  hello-bundled
warning [hello-bundled]:  3309528 extra bytes at beginning or within zipfile
  (attempting to process anyway)
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  11-25-2018 16:13   templates/
       14  11-05-2018 21:31   templates/hello.tmpl
---------                     -------
       14                     2 files
```

ここで `zip -A` (`--adjust-sfx`) するとオフセット値を調整することができる。

```
$ zip -A hello-bundled
Zip entry offsets appear off by 3309528 bytes - correcting...
```

調整が済めばself-extractingなZIPファイルになるため成功する。

```
$ ./hello-bundled 
2018/11/25 17:17:12 assets: /var/folders/xt/6z9sk1dx1d734ltxxst16_h00000gn/T/hello-bundled725779200
Hello, World!
```


# おわりに

ZIPを用いたリソースの埋め込み方法を紹介し、 `archive/zip` を用いたリソース埋め込みの例を解説した。

実際にこの埋め込み方法を使うには `go run` や `go test` がそのままでは動かない。また、リソースを差し替えるにはビルドの手順をやり直す必要がある。

シンプルなインタフェースで `go run` 時や `go test` 時のフォールバックと環境変数を用いたリソースの差し替えに対応した [assets](https://github.com/ichiban/assets) というライブラリを作ったので試してみてほしい。

[^wtf-is-sfx-stub]: SFXスタブとは何か？ということについて書かれた仕様書等は見つけられなかった

