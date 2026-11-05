---
title: Query in v2
date: 2026-11-04T00:00:00+09:00
---

In v1, [I modeled my query API for `ichiban/prolog` after `database/sql`](/post/query-in-v1-ichiban-prolog/). As I'm working on v2, I'm experimenting with an improved query API with newer Go features.


With recent Go features, including generic methods introduced in the newest Go 1.27, the query API can be as simple as a `for range` loop over `Query()`.

```go
for r, err := range p.Query[struct {
	Who prolog.Atom
}](ctx, `mortal(Who).`) {
	if err != nil {
		panic(err)
	}
	
	fmt.Printf("Who = %s\n", r.Who) // ==> Who = socrates
}
```

Compared with v1, which requires `Query()`, `Next()`, `Scan()`, `Err()`, and `Close()`, this is considerably simpler.

```go
sols, err := p.Query(`mortal(Who).`)
if err != nil {
	panic(err)
}
defer sols.Close()

for sols.Next() {
	var s struct {
		Who string
	}
	if err := sols.Scan(&s); err != nil {
		panic(err)
	}

	fmt.Printf("Who = %s\n", s.Who) // ==> Who = socrates
}
if err := sols.Err(); err != nil {
	panic(err)
}
```

The new standard iterator abstraction `iter.Seq2[K, V]`, introduced in Go 1.23, does the heavy lifting. It allows `Query()` to hide the explicit calls to `Next()`, `Err()`, and `Close()` from the caller. Together with range-over-function support, it lets callers iterate over results using a single `for range` loop.

Generics, introduced in Go 1.18 and extended to support generic methods in 1.27, allow `Query()` to return results in a specified Go type, eliminating the need for an explicit `Scan()`.

The goal hasn't changed since v1: to make Prolog feel natural to Go developers. Go itself has simply given me better tools to achieve it.
