---
type: post
title: Goで別パッケージの関数呼び出しは禁止すべき
date: 2019-12-11
hero: "/avoid-external-function-calls/leone-venter-mTkXSSScrzw-unsplash.jpg"
tags:
  - Go
  - StaticAnalysis
  - testing
authors:
  - Yutaka Ichibangase
---
# はじめに

Goのテストにおいて、別パッケージの関数呼び出しが問題となることを示し、その回避策とそれを徹底するための静的解析ツールを紹介する。

# 別パッケージの関数呼び出し

以下のプログラムで `main` をどうやってテストすればいいだろうか？例えば当日のお昼12:00に0日前だと表示されるケースをどう書けるだろうか。

```go
package main

import (
        "fmt"
        "time"
)

var date = must(time.Parse(time.RFC3339, "2019-12-20T00:00:00+09:00"))

func main() {
        d := date.Sub(time.Now()).Hours() / 24
        fmt.Printf("『スター・ウォーズ／スカイウォーカーの夜明け』まであと%d日\n", int(d))
}

func must(t time.Time, err error) time.Time {
        if err != nil {
                panic(err)
        }
        return t
}
```

この例のように、 `time.Now` や `fmt.Printf` のような別パッケージの関数の呼び出しがあるとテストできない[^seam][^monkey]。テストできないので、別パッケージの関数呼び出しは禁止すべき。

## 関数の変数を経由した呼び出し

一方、別パッケージの関数を変数に代入した場合、呼び出される関数はテスト時に置き換えることができる。

```go
package main

import (
	"fmt"
	"time"
)

var (
	timeParse         = time.Parse
	timeDurationHours = time.Duration.Hours
	timeTimeSub       = time.Time.Sub
	timeNow           = time.Now
	fmtPrintf         = fmt.Printf
)

var date = must(timeParse(time.RFC3339, "2019-12-20T00:00:00+09:00"))

func main() {
	d := timeDurationHours(timeTimeSub(date, timeNow())) / 24
	fmtPrintf("『スター・ウォーズ／スカイウォーカーの夜明け』まであと%d日\n", int(d))
}

func must(t time.Time, err error) time.Time {
	if err != nil {
		panic(err)
	}
	return t
}
```

そのため、容易にテストできる。

```go
package main

import (
	"testing"
	"time"

	"github.com/stretchr/testify/assert"
)

func Test_main(t *testing.T) {
	t.Run("当日12時は0日前", func(t *testing.T) {
		assert := assert.New(t)

		now, err := time.Parse(time.RFC3339, "2019-12-20T12:00:00+09:00")
		assert.NoError(err)

		n := timeNow
		defer func() { timeNow = n }()
		timeNow = func() time.Time {
			return now
		}

		called := false
		p := fmtPrintf
		defer func() { fmtPrintf = p }()
		fmtPrintf = func(format string, a ...interface{}) (int, error) {
			assert.Equal("『スター・ウォーズ／スカイウォーカーの夜明け』まであと%d日\n", format)
			assert.Equal([]interface{}{0}, a)

			called = true
			return 0, nil
		}

		main()

		assert.True(called)
	})
}
```

別パッケージの関数呼び出しをすべて禁止し、代わりに関数の変数を使った呼び出しにするべきだ。

# 静的解析ツールでの検知

別パッケージの関数呼び出しを検出する[静的解析ツールseams](https://github.com/ichiban/seams)を作ったので役立ててほしい。

```shell-session
$ go get github.com/ichiban/seams/cmd/seams
```

これを使えば、最初の例に対して以下のレポートが得られる。

```shell-session
$ seams ./...
/Users/ichiban/src/swuntestable/main.go:8:17: untestable function/method call: time.Parse
/Users/ichiban/src/swuntestable/main.go:11:7: untestable function/method call: (time.Duration).Hours
/Users/ichiban/src/swuntestable/main.go:11:7: untestable function/method call: (time.Time).Sub
/Users/ichiban/src/swuntestable/main.go:11:16: untestable function/method call: time.Now
/Users/ichiban/src/swuntestable/main.go:12:2: untestable function/method call: fmt.Printf
```


# おわりに

別パッケージの関数呼び出しが抱える問題を示し、関数の変数を使った回避策と静的解析ツールを紹介した。

冗談で書いてると思われるかもしれないが、けっこう本気でこういうのやったほうがいいんじゃないかと思ってるし、「Goのコードはこう書くべき」という主張はどんどん静的解析ツールにしていくといいんじゃないだろうか。

[^seam]: Working Effectively with Legacy Codeではseamという概念が紹介されており、テスト対象自体に変更を加えることなくプログラムの動作を変えられる箇所だと定義されている。Goの関数呼び出しは（interfaceのメソッドでない限り）seamでない。

[^monkey]: モンキーパッチをするという選択肢も技術的に無くはない。 [`bou.ke/monkey`](https://github.com/bouk/monkey) を使うことでできるが、[ライセンスに使ってはいけないと書いてある](https://github.com/bouk/monkey/blob/master/LICENSE.md)。

