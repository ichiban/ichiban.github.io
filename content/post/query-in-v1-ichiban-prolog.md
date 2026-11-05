---
title: Query in v1
date: 2026-11-03T00:00:00+09:00
---

`ichiban/prolog` is a Go library and my experiment in placing Prolog into the Go ecosystem.

By placement, I don't just mean it's a Prolog implementation written in Go or a Prolog implementation that you can access from Go. I mean it should feel as if it had always been part of Go.

For querying, I modeled the API after `database/sql`, Go's standard database interface in the standard library.

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

This is almost identical to SQL querying in Go:

```go
rows, err := db.Query(`SELECT * FROM mortals`)
if err != nil {
	panic(err)
}
defer rows.Close()

for rows.Next() {
	var s struct {
		Who string
	}
	if err := rows.Scan(&s.Who); err != nil {
		panic(err)
	}

	fmt.Printf("Who = %s\n", s.Who) // ==> Who = socrates
}
if err := rows.Err(); err != nil {
	panic(err)
}
```

This makes the API familiar to Go developers and easier to integrate it into their project.

But is the `database/sql`-like interface perfect for Prolog? There are two things to consider.

First, the `database/sql` API that I modeled mine after isn't even considered perfect by Go developers.
My `Scan()` is slightly more powerful because it maps query variable names to the corresponding struct field. This enhancement comes from [sqlx](https://github.com/jmoiron/sqlx/), an extension to the standard `database/sql`. sqlx's `StructScan()` maps each column to the corresponding struct field.
`database/sql` is a solid API but very few Go developers use it as is. Most of them use it through a wrapper like `sqlx` or ORM like [GORM](https://gorm.io/index.html)

Second, Go has gained new features that might have changed the design decisions.
The Go language is slowly but steadily evolving. Go introduced a new standard iterator abstraction `iter` in Go 1.23, generics in Go 1.18, and generic methods in Go 1.27.
These features open up new possibilities for designing an API tailored for Prolog.
