---
type: post
title: 一発でセラーの中身をぶっこぬく（あるいはgoaでHAL+JSON）
date: 2017-12-01
hero: "/hal-json-in-goa/hermes-rivera-aK6WGqxyHFw-unsplash.jpg"
tags:
  - Go
  - goa
  - Hypermedia
  - proxy
  - JSON
authors:
  - Yutaka Ichibangase
---
# はじめに

Goでweb APIを作る際には[goa](https://goa.design/)が便利だが、デザイン時にひと工夫することでAPIの柔軟性が増す。ここではgoaの例としてしばしば用いられる[ワインセラーのAPI](https://github.com/goadesign/goa-cellar)を基に、[HAL+JSON](https://tools.ietf.org/html/draft-kelly-json-hal-08)を取り入れることでセラーの中身すべてを１リクエストで取得する。

# セラーの中身をすべて

オリジナルのデザインのAPIでセラーの中身すべて、すなわち全アカウントの全ボトルの情報を取得するにはどうすればいいだろうか？

これは `/cellar/accounts` を起点に各アカウントからアカウント毎のボトルの一覧、ボトルの詳細へと辿って行くことで実現できる。

```shell-session
$ curl http://localhost:8081/cellar/accounts
[{"href":"/cellar/accounts/1","id":1,"name":"account 1"},{"href":"/cellar/accounts/2","id":2,"name":"account 2"}]
$ curl http://localhost:8081/cellar/accounts/1
{"created_at":"0001-01-01T00:00:00Z","created_by":"","href":"/cellar/accounts/1","id":1,"name":"account 1"}
$ curl http://localhost:8081/cellar/accounts/2
{"created_at":"0001-01-01T00:00:00Z","created_by":"","href":"/cellar/accounts/2","id":2,"name":"account 2"}
$ curl http://localhost:8081/cellar/accounts/1/bottles
[{"account":{"href":"/cellar/accounts/1","id":1,"name":"account 1"},"href":"/cellar/accounts/1/bottles/100","id":100,"links":{"account":{"href":"/cellar/accounts/1","id":1}},"name":"Number 8","rating":4,"varietal":"Merlot","vineyard":"Asti Winery","vintage":2012},{"account":{"href":"/cellar/accounts/1","id":1,"name":"account 1"},"href":"/cellar/accounts/1/bottles/101","id":101,"links":{"account":{"href":"/cellar/accounts/1","id":1}},"name":"Mourvedre","rating":3,"varietal":"Mourvedre","vineyard":"Rideau","vintage":2012},{"account":{"href":"/cellar/accounts/1","id":1,"name":"account 1"},"href":"/cellar/accounts/1/bottles/102","id":102,"links":{"account":{"href":"/cellar/accounts/1","id":1}},"name":"Blue's Cuvee","rating":5,"varietal":"Cabernet Franc with Merlot, Malbec, Cabernet Sauvignon and Syrah","vineyard":"Longoria","vintage":2012}]
$ curl http://localhost:8081/cellar/accounts/2/bottles
[{"account":{"href":"/cellar/accounts/2","id":2,"name":"account 2"},"href":"/cellar/accounts/2/bottles/200","id":200,"links":{"account":{"href":"/cellar/accounts/2","id":2}},"name":"Blackstone Merlot","rating":3,"varietal":"Merlot","vineyard":"Blackstone","vintage":2012},{"account":{"href":"/cellar/accounts/2","id":2,"name":"account 2"},"href":"/cellar/accounts/2/bottles/201","id":201,"links":{"account":{"href":"/cellar/accounts/2","id":2}},"name":"Wild Horse","rating":4,"varietal":"Pinot Noir","vineyard":"Wild Horse","vintage":2010}]
$ curl http://localhost:8081/cellar/accounts/1/bottles/100
{"account":{"href":"/cellar/accounts/1","id":1,"name":"account 1"},"href":"/cellar/accounts/1/bottles/100","id":100,"links":{"account":{"href":"/cellar/accounts/1","id":1}},"name":"Number 8","rating":4,"varietal":"Merlot","vineyard":"Asti Winery","vintage":2012}
$ curl http://localhost:8081/cellar/accounts/1/bottles/101
{"account":{"href":"/cellar/accounts/1","id":1,"name":"account 1"},"href":"/cellar/accounts/1/bottles/101","id":101,"links":{"account":{"href":"/cellar/accounts/1","id":1}},"name":"Mourvedre","rating":3,"varietal":"Mourvedre","vineyard":"Rideau","vintage":2012}
$ curl http://localhost:8081/cellar/accounts/1/bottles/102
{"account":{"href":"/cellar/accounts/1","id":1,"name":"account 1"},"href":"/cellar/accounts/1/bottles/102","id":102,"links":{"account":{"href":"/cellar/accounts/1","id":1}},"name":"Blue's Cuvee","rating":5,"varietal":"Cabernet Franc with Merlot, Malbec, Cabernet Sauvignon and Syrah","vineyard":"Longoria","vintage":2012}
$ curl http://localhost:8081/cellar/accounts/2/bottles/200
{"account":{"href":"/cellar/accounts/2","id":2,"name":"account 2"},"href":"/cellar/accounts/2/bottles/200","id":200,"links":{"account":{"href":"/cellar/accounts/2","id":2}},"name":"Blackstone Merlot","rating":3,"varietal":"Merlot","vineyard":"Blackstone","vintage":2012}
$ curl http://localhost:8081/cellar/accounts/2/bottles/201
{"account":{"href":"/cellar/accounts/2","id":2,"name":"account 2"},"href":"/cellar/accounts/2/bottles/201","id":201,"links":{"account":{"href":"/cellar/accounts/2","id":2}},"name":"Wild Horse","rating":4,"varietal":"Pinot Noir","vineyard":"Wild Horse","vintage":2010}
```

しかし、この方法では必要となるリクエストの数が10と多い。

# HAL+JSON

HAL+JSONはJSONに関連するリソースへのリンクを表す`_links`プロパティや、リソースの埋め込みを表す`_embed`プロパティを定義する。

例えばあるアカウントのボトル一覧のリソースは自身へのリンク`self`、その所有者であるアカウントへのリンク`account`、要素である各ボトルへのリンク`members`を`_links`プロパティに持つことで表現される。

```json
{
  "_links": {
    "account": {
      "href": "/accounts/1"
    },
    "members": [
      {
        "href": "/accounts/1/bottles/100"
      },
      {
        "href": "/accounts/1/bottles/101"
      },
      {
        "href": "/accounts/1/bottles/102"
      }
    ],
    "self": {
      "href": "/accounts/1/bottles"
    }
  }
}
```

goaでHAL+JSONを返すには、各リソースのメディアタイプに対して、リンクを表すタイプ（この場合は`BottleCollectionLinks`）を定義する。

```go
var BottleCollection = MediaType("application/vnd.bottle-collection+json", func() {
	Attributes(func() {
		Attribute("_links", BottleCollectionLinks)

		Required("_links")
	})

	View("default", func() {
		Attribute("_links")
	})
})
```

```go
var BottleCollectionLinks = Type("BottleCollectionLinks", func() {
	Attribute("self", HALLink)
	Attribute("account", HALLink)
	Attribute("members", ArrayOf(HALLink))

	Required("self", "account", "members")
})
```

```go
var HALLink = Type("HALLink", func() {
	Attribute("href", String)

	Required("href")
})
```

HAL+JSON化したワインセラーのAPIのソースコードは https://github.com/ichiban/cellar にある。このデザインではオリジナルのAPI同様、すべての情報を一度に取得するというユースケースは想定していない。

# セラーの中身をすべて（１リクエストで）

[jesiというリバース・プロキシ](https://github.com/ichiban/jesi)がある。なぜあるかというと作ったからだが、jesiを経由させることで任意のリンクされたリソースを埋め込むことができる。

```shell-session
$ jesi -port 8080 -backend http://localhost:8081/accounts
INFO[0000] Added a backend                               backend="http://localhost:8081/accounts" queue=healthy
INFO[0000] Start a server                                max=67108864 node=_6a40e7d5-e66b-4cfd-a117-d458effda69a port=8080 verbose=false version=
```

これを使えば、アカウント一覧の取得時に、各アカウント`member`のボトル一覧`bottles`の各ボトル`members`を埋め込むよう指定することですべての情報を一度に取得できる。

```shell-session
$ curl -H 'With: members.bottles.members' http://localhost:8080/accounts | jq .
{
  "_embedded": {
    "members": [
      {
        "_embedded": {
          "bottles": {
            "_embedded": {
              "members": [
                {
                  "_links": {
                    "collection": {
                      "href": "/accounts/1/bottles"
                    },
                    "self": {
                      "href": "/accounts/1/bottles/100"
                    }
                  },
                  "id": 100,
                  "name": "Number 8",
                  "rating": 4,
                  "varietal": "Merlot",
                  "vineyard": "Asti Winery",
                  "vintage": 2012
                },
                {
                  "_links": {
                    "collection": {
                      "href": "/accounts/1/bottles"
                    },
                    "self": {
                      "href": "/accounts/1/bottles/101"
                    }
                  },
                  "id": 101,
                  "name": "Mourvedre",
                  "rating": 3,
                  "varietal": "Mourvedre",
                  "vineyard": "Rideau",
                  "vintage": 2012
                },
                {
                  "_links": {
                    "collection": {
                      "href": "/accounts/1/bottles"
                    },
                    "self": {
                      "href": "/accounts/1/bottles/102"
                    }
                  },
                  "id": 102,
                  "name": "Blue's Cuvee",
                  "rating": 5,
                  "varietal": "Cabernet Franc with Merlot, Malbec, Cabernet Sauvignon and Syrah",
                  "vineyard": "Longoria",
                  "vintage": 2012
                }
              ]
            },
            "_links": {
              "account": {
                "href": "/accounts/1"
              },
              "members": [
                {
                  "href": "/accounts/1/bottles/100"
                },
                {
                  "href": "/accounts/1/bottles/101"
                },
                {
                  "href": "/accounts/1/bottles/102"
                }
              ],
              "self": {
                "href": "/accounts/1/bottles"
              }
            }
          }
        },
        "_links": {
          "bottles": {
            "href": "/accounts/1/bottles"
          },
          "collection": {
            "href": "/accounts"
          },
          "self": {
            "href": "/accounts/1"
          }
        },
        "created_at": "2017-11-18T09:34:18Z",
        "created_by": "",
        "id": 1,
        "name": "account 1"
      },
      {
        "_embedded": {
          "bottles": {
            "_embedded": {
              "members": [
                {
                  "_links": {
                    "collection": {
                      "href": "/accounts/2/bottles"
                    },
                    "self": {
                      "href": "/accounts/2/bottles/200"
                    }
                  },
                  "id": 200,
                  "name": "Blackstone Merlot",
                  "rating": 3,
                  "varietal": "Merlot",
                  "vineyard": "Blackstone",
                  "vintage": 2012
                },
                {
                  "_links": {
                    "collection": {
                      "href": "/accounts/2/bottles"
                    },
                    "self": {
                      "href": "/accounts/2/bottles/201"
                    }
                  },
                  "id": 201,
                  "name": "Wild Horse",
                  "rating": 4,
                  "varietal": "Pinot Noir",
                  "vineyard": "Wild Horse",
                  "vintage": 2010
                }
              ]
            },
            "_links": {
              "account": {
                "href": "/accounts/2"
              },
              "members": [
                {
                  "href": "/accounts/2/bottles/200"
                },
                {
                  "href": "/accounts/2/bottles/201"
                }
              ],
              "self": {
                "href": "/accounts/2/bottles"
              }
            }
          }
        },
        "_links": {
          "bottles": {
            "href": "/accounts/2/bottles"
          },
          "collection": {
            "href": "/accounts"
          },
          "self": {
            "href": "/accounts/2"
          }
        },
        "created_at": "2017-11-18T09:34:18Z",
        "created_by": "",
        "id": 2,
        "name": "account 2"
      }
    ]
  },
  "_links": {
    "members": [
      {
        "href": "/accounts/1"
      },
      {
        "href": "/accounts/2"
      }
    ],
    "self": {
      "href": "/accounts"
    }
  }
}
```

# おわりに

goaでHAL+JSONを返す方法を示し、それがAPIに柔軟性を与え、ワインセラーのAPIの例ですべての情報を１リクエストで取得できることを示した。

ところで飲酒は喫煙と並んで現代社会が克服すべき悪習である。

