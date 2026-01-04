---
type: post
title: Goのテストでヘルパー関数に t.Helper() を忘れない
date: 2019-12-07
hero: "/dont-forget-t-helper/fabrizio-frigeni-vHLj6G8xLhU-unsplash.jpg"
tags:
  - Go
  - testing
  - StaticAnalysis
authors:
  - Yutaka Ichibangase
---
# はじめに

Goのテストにおいて、[ヘルパー関数は `t.Helper()` を呼ぶことでヘルパー関数だとマークできる](https://golang.org/pkg/testing/#T.Helper)。

この記事では、ヘルパー関数としてマークしないとどういった問題があるか、マークするとどう問題が改善されるか、またマークし忘れを防ぐ方法について説明する。

# ヘルパー関数としてマークしないと

ヘルパー関数 `testChdir()` を用いたテストについて考える。この関数は第２引数に存在しないパスを渡すと `t.Fatalf()` が呼ばれる。 `TestFoo()` は存在しないパスで `testChdir()` を呼ぶため失敗する。

しかし `go test` の出力で失敗の箇所は `b_test.go:22` であり、これは `testChdir()` 内の `t.Fatalf("err: %s", err)` を指している。そのため、なぜテストが失敗しているのかを理解するのにあまり有用ではない。

```go
package b

import (
	"os"
	"testing"
)

func TestFoo(t *testing.T) {
	defer testChdir(t, "/this/directory/does/not/exist")()

	// ...
}

// https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=30
func testChdir(t *testing.T, dir string) func() {
	old, err := os.Getwd()
	if err != nil {
		t.Fatalf("err: %s", err)
	}

	if err := os.Chdir(dir); err != nil {
		t.Fatalf("err: %s", err) // b_test.go:22
	}

	return func() { os.Chdir(old) }
}
```

```shell-session
$ go test
--- FAIL: TestFoo (0.00s)
    b_test.go:22: err: chdir /this/directory/does/not/exist: no such file or directory
FAIL
exit status 1
FAIL    github.com/ichiban/thelper/testdata/src/b       0.006s
```

# ヘルパー関数としてマークすると

一方、 `testChdir()` の冒頭で `t.Helper()` を呼ぶと `go test` の結果は失敗の箇所を `b_test.go:9` と表示し、これは `TestFoo()` 内の `defer testChdir(t, "/this/directory/does/not/exist")()` を指しているため失敗の原因がわかりやすい。

テストのヘルパー関数では忘れずに `t.Helper()` を呼ぶべきだ。

```go
package b

import (
	"os"
	"testing"
)

func TestFoo(t *testing.T) {
	defer testChdir(t, "/this/directory/does/not/exist")() // b_test.go:9

	// ...
}

// https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=30
func testChdir(t *testing.T, dir string) func() {
	t.Helper()

	old, err := os.Getwd()
	if err != nil {
		t.Fatalf("err: %s", err)
	}

	if err := os.Chdir(dir); err != nil {
		t.Fatalf("err: %s", err)
	}

	return func() { os.Chdir(old) }
}
```

```shell-session
$ go test
--- FAIL: TestFoo (0.00s)
    b_test.go:9: err: chdir /this/directory/does/not/exist: no such file or directory
FAIL
exit status 1
FAIL    github.com/ichiban/thelper/testdata/src/b       0.006s
```

# マークし忘れを防ぐ

`t.Helper()` の呼び忘れを防ぐには人力ではどうしても漏れがでる。そのため、 [`thelper`](https://github.com/ichiban/thelper) という静的解析ツールを作った。これを使うとマークし忘れたヘルパー関数が見つかる。

```shell-session
$ go get github.com/ichiban/thelper/cmd/thelper
```

```shell-session
$ thelper ./...
/Users/ichiban/src/thelper/testdata/src/b/b_test.go:15:1: unmarked test helper: call t.Helper()
```

# おわりに

ヘルパー関数をマークしないとどういう問題があるか、マークすることでどう改善されるかについて説明し、マークし忘れを防止する静的解析ツールを紹介した。

コードを書く上で気をつけなければいけないが見落としがちなことは静的解析ツールにしてしまうといいんじゃないだろうか。

