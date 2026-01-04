---
type: post
title: html/templateで共通のボイラープレートを使いまわして異なるコンテンツを出す
date: 2017-08-27
hero: "/html-template-boilerplate/neven-krcmarek-0TH1H1rq_eY-unsplash.jpg"
tags:
  - Go
authors:
  - Yutaka Ichibangase
---
# やりたいこと

Goでwebアプリを書く際にサーバサイドで `html/template` を使って複数のコンテンツをHTMLにレンダリングするが、ボイラープレートを抜き出して共通化したい。

# やりかた

あらかじめ、初期化時にすべてのテンプレートをパースしておく。このとき、ボイラープレート中で `"content"` を未定義としておき、コンテンツでも定義しないでおく。レンダリング時、コンテンツ毎に `"content"` をそのコンテンツのパース済みのテンプレートへの別名として定義しレンダリングする。

# 具体例

ボイラープレートを `layout.html` 、異なる２つのコンテンツとして `foo.html` と `bar.html` を使った例を考える。

 `foo.html` と `bar.html` は個別のコンテンツを含むが、 `<html>` や `<body>` といったボイラープレートが無い不完全なテンプレートであり、一方 `layout.html` は `"content"` が未定義なのでこのままではレンダリングできないテンプレートである。

これら３つのテンプレートを含む `*template.Template` をクローンし、 `"content"` を `foo.html` または `bar.html` の `*parse.Tree` に紐付けることで、ボイラープレートを使ったテンプレートが得られる。

https://play.golang.org/p/yLlFoOe2ET

```go
package main

import (
	"html/template"
	"os"
)

// 実際には `var ts = template.Must(template.ParseGlob("./views/*.html"))` とかでやりそう
var ts *template.Template
func init() {
	ts = template.Must(template.New("layout.html").Parse(`
<html>
<body>
  {{ template "content" . }}
</body>
</html>
`))
	ts = template.Must(ts.New("foo.html").Parse(`<div>this is foo</div>`))
	ts = template.Must(ts.New("bar.html").Parse(`<div>this is bar</div>`))
}

func main() {
	t := template.Must(ts.Clone())
	t = template.Must(t.AddParseTree("content", t.Lookup("foo.html").Tree))
	if err := t.ExecuteTemplate(os.Stdout, "layout.html", nil); err != nil {
		panic(err)
	}
	
	t = template.Must(ts.Clone())
	t = template.Must(t.AddParseTree("content", t.Lookup("bar.html").Tree))
	if err := t.ExecuteTemplate(os.Stdout, "layout.html", nil); err != nil {
		panic(err)
	}
}
```

実行すると、共通のボイラープレートを使った完全な `foo.html` と `bar.html` がそれぞれ出力される。

```

<html>
<body>
  <div>this is foo</div>
</body>
</html>

<html>
<body>
  <div>this is bar</div>
</body>
</html>

```

