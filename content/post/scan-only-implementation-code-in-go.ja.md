---
type: post
title: Goの静的解析ツールで実装コードだけを走査する
date: 2019-12-02
hero: "/scan-only-implementation-code-in-go/niko-photos-tGTVxeOr_Rs-unsplash.jpg"
tags:
  - Go
  - StaticAnalysis
authors:
  - Yutaka Ichibangase
---
# はじめに

Goの静的解析ツールは `golang.org/x/tools/go/analysis` を使うことで開発でき、構文木を走査するのに `golang.org/x/tools/go/analysis/passes/inspect` が使える。しかし実装コード以外も走査されるため、実装コードに焦点を当てた静的解析ツールを作る際に邪魔になる。

ここでは `Nodes` を使った走査を紹介し、それを用いてテストファイルとジェネレータで生成されたファイルを除外する方法を説明する。

# Preorderを使った走査

定義した関数名を一覧する静的解析ツールについて考える。 `Preorder` を使うとかなりそれらしいものがシンプルに書ける。

```go
package funcdecl

import (
	"go/ast"

	"golang.org/x/tools/go/analysis"
	"golang.org/x/tools/go/analysis/passes/inspect"
	"golang.org/x/tools/go/ast/inspector"
)

var Analyzer = &analysis.Analyzer{
	Name:     "funcdecl",
	Doc:      `find function declarations`,
	Requires: []*analysis.Analyzer{inspect.Analyzer},
	Run:      run,
}

func run(pass *analysis.Pass) (interface{}, error) {
	inspect := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)

	inspect.Preorder([]ast.Node{
		(*ast.FuncDecl)(nil), // 関心のあるノードの種類の値を列挙する（この例では関数定義のみ）
	}, func(node ast.Node) {
		f := node.(*ast.FuncDecl)
		pass.Reportf(f.Pos(), `found %s`, f.Name)
	})

	return nil, nil
}
```

しかしこれには問題があり、テストファイルとジェネレータで生成されたファイルも対象となってしまう。以下の例で `a_test.go` はテストファイル、最後のキャッシュはビルド由来の生成されたファイルである。これらを除外したい。

```shell-session
$ funcdecl ./...
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:3:1: found main
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:7:1: found Foo
/Users/ichiban/src/funcdecl/testdata/src/a/a_test.go:7:1: found TestFoo
/Users/ichiban/Library/Caches/go-build/be/bef6acc9728a13887659154cd4c2d9b8fb8e3d8d505ef63aeba17380c0b75212-d:34:1: found init
/Users/ichiban/Library/Caches/go-build/be/bef6acc9728a13887659154cd4c2d9b8fb8e3d8d505ef63aeba17380c0b75212-d:40:1: found main
```

# Nodesを使った走査

 `Nodes` は `Preorder` よりもやや複雑だがより細かい制御ができ、特定のサブツリーを除外することができる。

`Preorder` との違いはコールバック関数にあり、 ひとつのノードに対して２回呼ばれる。一度目は子ノードを処理する前で、引数の `push` には `true` が渡される。また、戻り値として `false` を返すと子ノードをルートとするサブツリーを処理しなくなる。二度目は子ノード（とそれをルートとするサブツリー）を処理した後で、 `push` として `false` が渡され、戻り値は無視される。

```go
package funcdecl

import (
	"go/ast"

	"golang.org/x/tools/go/analysis"
	"golang.org/x/tools/go/analysis/passes/inspect"
	"golang.org/x/tools/go/ast/inspector"
)

var Analyzer = &analysis.Analyzer{
	Name:     "funcdecl",
	Doc:      `find function declarations`,
	Requires: []*analysis.Analyzer{inspect.Analyzer},
	Run:      run,
}

func run(pass *analysis.Pass) (interface{}, error) {
	inspect := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)

	inspect.Nodes([]ast.Node{
		(*ast.FuncDecl)(nil), // 関心のあるノードの種類の値を列挙する（この例では関数定義のみ）
	}, func(n ast.Node, push bool) bool {
		if !push { // 子ノードを処理する前にだけ関心がある
			return false
		}

		f := n.(*ast.FuncDecl)
		pass.Reportf(f.Pos(), `found %s`, f.Name)

		return false // 関数定義はネストしないのでサブツリーには関心がない
	})

	return nil, nil
}
```

これは前述の `Preorder` を用いたものと同じ結果になる。

```shell-session
$ funcdecl ./...
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:3:1: found main
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:7:1: found Foo
/Users/ichiban/src/funcdecl/testdata/src/a/a_test.go:7:1: found TestFoo
/Users/ichiban/Library/Caches/go-build/be/bef6acc9728a13887659154cd4c2d9b8fb8e3d8d505ef63aeba17380c0b75212-d:34:1: found init
/Users/ichiban/Library/Caches/go-build/be/bef6acc9728a13887659154cd4c2d9b8fb8e3d8d505ef63aeba17380c0b75212-d:40:1: found main
```

これを用いて問題のファイルを除外していく。

# テストファイルの除外

テストファイルはファイル名の末尾が `_test.go` なのでそれを条件に除外できる。

```go
package funcpattern

import (
	"go/ast"
	"regexp"
	"strings"

	"golang.org/x/tools/go/analysis"
	"golang.org/x/tools/go/analysis/passes/inspect"
	"golang.org/x/tools/go/ast/inspector"
)

var Analyzer = &analysis.Analyzer{
    Name:     "funcdecl",
    Doc:      `find function declarations`,
    Requires: []*analysis.Analyzer{inspect.Analyzer},
    Run:      run,
}

func run(pass *analysis.Pass) (interface{}, error) {
	inspect := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)

	inspect.Nodes([]ast.Node{
		(*ast.File)(nil), // 関数定義だけでなくファイルにも関心が出た
		(*ast.FuncDecl)(nil),
	}, func(n ast.Node, push bool) bool {
		if !push {
			return false
		}

		switch n := n.(type) {
		case *ast.File:
			f := pass.Fset.File(n.Pos())
			return !strings.HasSuffix(f.Name(), "_test.go") // 末尾が `_test.go` であるサブツリーには関心がない
		case *ast.FuncDecl:
			pass.Reportf(f.Pos(), `found %s`, n.Name)
			return false
		default:
			panic(n)
		}
	})

	return nil, nil
}
```

テストファイルが除外された。

```shell-session
$ funcdecl ./...
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:3:1: found main
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:7:1: found Foo
/Users/ichiban/Library/Caches/go-build/be/bef6acc9728a13887659154cd4c2d9b8fb8e3d8d505ef63aeba17380c0b75212-d:34:1: found init
/Users/ichiban/Library/Caches/go-build/be/bef6acc9728a13887659154cd4c2d9b8fb8e3d8d505ef63aeba17380c0b75212-d:40:1: found main
```


# ジェネレータで生成されたファイルの除外

ジェネレータで生成されたファイルは、コメントに `DO NOT EDIT` と含まれるように、利用者としてはそのコードを変更すべきでない。そのため静的解析ツールのレポートは邪魔になる。

実はこのコメント `// Code generated * DO NOT EDIT.` は[ファイルが生成されたものかどうかを判別するのに使ってよいことになっている](https://golang.org/s/generatedcode)。

```go
package funcdecl

import (
	"go/ast"
	"regexp"
	"strings"

	"golang.org/x/tools/go/analysis"
	"golang.org/x/tools/go/analysis/passes/inspect"
	"golang.org/x/tools/go/ast/inspector"
)

var Analyzer = &analysis.Analyzer{
	Name:     "funcdecl",
	Doc:      `find function declarations`,
	Requires: []*analysis.Analyzer{inspect.Analyzer},
	Run:      run,
}

func run(pass *analysis.Pass) (interface{}, error) {
	inspect := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)

	inspect.Nodes([]ast.Node{
		(*ast.File)(nil),
		(*ast.FuncDecl)(nil),
	}, func(n ast.Node, push bool) bool {
		if !push {
			return false
		}

		switch n := n.(type) {
		case *ast.File:
			f := pass.Fset.File(n.Pos())
			if strings.HasSuffix(f.Name(), "_test.go") {
				return false
			}

			return !generated(n) // ジェネレータで生成されたファイルのサブツリーには関心がない
		case *ast.FuncDecl:
			pass.Reportf(n.Pos(), `found %s`, n.Name)
			return false
		default:
			panic(n)
		}
	})

	return nil, nil
}

// https://github.com/golang/go/issues/13560#issuecomment-288457920
var pattern = regexp.MustCompile(`^// Code generated .* DO NOT EDIT\.$`)

// ファイルのどこかに生成されたことを表すコメントがある
func generated(f *ast.File) bool {
	for _, c := range f.Comments {
		for _, l := range c.List {
			if pattern.MatchString(l.Text) {
				return true
			}
		}
	}
	return false
}
```

ジェネレータで生成されたファイルも除外された。

```shell-session
$ funcdecl ./...
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:3:1: found main
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:7:1: found Foo
```

# おわりに

`Nodes` を使った走査を紹介し、それを用いてテストファイルとジェネレータで生成されたファイルを除外する方法を説明した。

とはいえ、毎回明示的にこれらのファイルを除外するのは面倒だ。あらかじめ除外していてくれるラッパー [`ichiban/prodinspect`](https://github.com/ichiban/prodinspect) を作ったので活用してほしい。これを使うと最初の `Preorder` の例とほぼ同じコードで意図した結果が得られる。

```go
package funcdecl

import (
	"go/ast"

	"golang.org/x/tools/go/analysis"

	"github.com/ichiban/prodinspect"
)

var Analyzer = &analysis.Analyzer{
	Name:     "funcdecl",
	Doc:      `find function declarations`,
	Requires: []*analysis.Analyzer{prodinspect.Analyzer},
	Run:      run,
}

func run(pass *analysis.Pass) (interface{}, error) {
	inspect := pass.ResultOf[prodinspect.Analyzer].(*prodinspect.Inspector)

	inspect.Preorder([]ast.Node{
		(*ast.FuncDecl)(nil),
	}, func(n ast.Node) {
		f := n.(*ast.FuncDecl)
		pass.Reportf(f.Pos(), `found %s`, f.Name)
	})

	return nil, nil
}
```

```shell-session
$ funcdecl ./...
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:3:1: found main
/Users/ichiban/src/funcdecl/testdata/src/a/a.go:7:1: found Foo
```

