---
type: post
title: リバース・プロキシはどのようなリクエスト／レスポンスのときにキャッシュするか？
date: 2017-06-23
hero: "/when-does-a-reverse-proxy-cache/fabio-issao-h9j5OJVhx68-unsplash.jpg"
tags:
  - reverse proxy
  - CDN
  - HTTP
  - Security
authors:
  - Yutaka Ichibangase
---
[メルカリでCDNのキャッシュに由来する情報流出](http://tech.mercari.com/entry/2017/06/22/204500)があった。CDNでキャッシュしているのはリバース・プロキシで、ちょっと前に[Goの練習を兼ねてリバース・プロキシを書いた](https://github.com/ichiban/jesi)ので解説してみる。

リバース・プロキシのキャッシュの挙動について考えるとき、以下の２点を切り離して考える必要がある：

- どのようなリクエスト／レスポンスのときにキャッシュするか？
- どのようなリクエストのときにキャッシュから返すか？

[メルカリのケース](http://tech.mercari.com/entry/2017/06/22/204500)では、そもそもキャッシュして欲しくない情報がキャッシュされ、かつそれがユーザへのレスポンスとして使われたという問題なので、ここでは前者の「どのようなリクエスト／レスポンスのときにキャッシュするか？」について考える。

リバース・プロキシはどのようなリクエスト／レスポンスのときにキャッシュするのか？
[RFC7234](https://tools.ietf.org/html/rfc7234#section-3)的には以下のすべての条件を満たすときにキャッシュするかもしれない。

- キャッシュ可能なメソッドある
- キャッシュ可能なステータスコードである
- **リクエスト**ヘッダの`Cache-Control`に`no-store`がない
- **レスポンス**ヘッダの`Cache-Control`に`no-store`や`private`がない
- **リクエスト**ヘッダに`Authorization`がない
- 以下のいずれかを満たす
  - **レスポンス**ヘッダに`Expires`がある、または
  - **レスポンス**ヘッダの`Cache-Control`に`max-age`がある、または
  - **レスポンス**ヘッダの`Cache-Control`に`s-maxage`がある、または
  - **レスポンス**ヘッダの`Cache-Control`においてCache Control Extensionsでキャッシュ可能だと指定されている、または
  - キャッシュ可能だと定義されているステータス・コード、または
  - **レスポンス**ヘッダの`Cache-Control`に`public`がある

[メルカリのケース](http://tech.mercari.com/entry/2017/06/22/204500)ではレスポンス・ヘッダに以下を指定していた。

```
Cache-Control: no-cache
Expires: Thu, 22 Jun 2017 08:58:21 GMT (アクセスの1秒前の時間)
```

上に書いたキャッシュの条件で、`Cache-Control: no-cache`は出てこない。[RFC7234](https://tools.ietf.org/html/rfc7234#section-5.2.1.4)では、`no-cache`はキャッシュされたレスポンスを使わないように指定するものだと書いてある。つまり、「どのようなリクエストのときにキャッシュから返すか？」の話だ。

意外なことに、古い日付の`Expires`があることはキャッシュされる原因になりうる。なぜなら、リバース・プロキシはアプリケーション・サーバ等の上流のサーバが反応しないときに、古い内容であると分かっていながらキャッシュからレスポンスを返すことがあるからだ。

では、どうすればキャッシュされなくなるか？上に書いたキャッシュの条件に当てはまらなくなればいい。具体的には**レスポンス**ヘッダに`Cache-Control: private`でよい。

