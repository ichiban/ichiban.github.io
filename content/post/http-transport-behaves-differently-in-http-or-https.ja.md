---
type: post
title: http.Transport はHTTPプロキシの先がHTTPかHTTPSかで挙動が違う
date: 2018-02-25
hero: "/http-transport-behaves-differently-in-http-or-https/joey-kyber-45FJgZMXCK8-unsplash.jpg"
tags:
  - Go
authors:
  - Yutaka Ichibangase
---
`http.Client` 等で内部的に使われている `http.Transport` はプロキシの利用を隠蔽するが、HTTPプロキシを利用しておりそのプロキシが `CONNECT` に対して `200 OK` 以外のステータスコードを返すとき、接続先がHTTPかHTTPSかで異なるふるまいをすることに気づいた

https://play.golang.org/p/8K_9FUxyU_r

```go
package main

import (
	"crypto/tls"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/http/httptest"
	"net/url"
	"regexp"
	"sync"
)

func main() {
	origin := origin()
	defer origin.Close()

	originTLS := originTLS()
	defer originTLS.Close()

	ok := okProxy()
	defer ok.Close()

	badGateway := badGatewayProxy()
	defer badGateway.Close()

	testCases := []struct {
		title string

		origin *httptest.Server
		proxy  *httptest.Server

		status int
		error  *regexp.Regexp
	}{
		{
			title: "オリジンがHTTP、プロキシがOK",

			origin: origin,
			proxy:  ok,

			status: http.StatusOK,
		},
		{
			title: "オリジンがHTTPS、プロキシがOK",

			origin: originTLS,
			proxy:  ok,

			status: http.StatusOK,
		},
		{
			title: "オリジンがHTTP、プロキシがBad Gateway",

			origin: origin,
			proxy:  badGateway,

			status: http.StatusBadGateway,
		},
		{
			title: "オリジンがHTTPS、プロキシがBad Gateway",

			origin: originTLS,
			proxy:  badGateway,

			error: regexp.MustCompile(`\AGet https://(?:.+): Bad Gateway\z`),
		},
	}

	for _, tc := range testCases {
		fmt.Printf("%s: ", tc.title)

		u, err := url.Parse(tc.proxy.URL)
		if err != nil {
			panic(err)
		}
		c := http.Client{
			Transport: &http.Transport{
				TLSClientConfig: &tls.Config{
					InsecureSkipVerify: true,
				},
				Proxy: http.ProxyURL(u),
			},
		}
		res, err := c.Get(tc.origin.URL)
		if tc.status != 0 {
			if res == nil {
				panic("res expected")
			}
			if tc.status != res.StatusCode {
				panic(res.StatusCode)
			}
			fmt.Printf("%dを返す\n", tc.status)
		}
		if tc.error != nil {
			if !tc.error.MatchString(err.Error()) {
				panic(err)
			}
			fmt.Printf("エラーになる\n")
		}
	}
}

func originTLS() *httptest.Server {
	return httptest.NewTLSServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Hello TLS")
	}))
}

func origin() *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Hello")
	}))
}

func okProxy() *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		up, err := net.Dial("tcp", r.Host)
		if err != nil {
			panic(err)
		}
		w.WriteHeader(http.StatusOK)
		hijacker, _ := w.(http.Hijacker)
		down, _, err := hijacker.Hijack()
		if err != nil {
			panic(err)
		}
		var wg sync.WaitGroup

		wg.Add(1)
		go func() {
			defer wg.Done()
			if _, err := io.Copy(up, down); err != nil {
				panic(err)
			}
		}()

		wg.Add(1)
		go func() {
			defer wg.Done()
			if _, err := io.Copy(down, up); err != nil {
				panic(err)
			}
		}()

		wg.Wait()
		if err := up.Close(); err != nil {
			panic(err)
		}
		if err := down.Close(); err != nil {
			panic(err)
		}
	}))
}

func badGatewayProxy() *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, "", http.StatusBadGateway)
	}))
}
```

```
オリジンがHTTP、プロキシがOK: 200を返す
オリジンがHTTPS、プロキシがOK: 200を返す
オリジンがHTTP、プロキシがBad Gateway: 502を返す
オリジンがHTTPS、プロキシがBad Gateway: エラーになる
```

また、HTTPSだったときはプロキシのアウトバウンド側のコネクションを閉じたときもエラーになる

