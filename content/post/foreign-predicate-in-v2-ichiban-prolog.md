---
title: Foreign predicate in v2
date: 2026-11-04T01:00:00+09:00
---

`ichiban/prolog` v1 was built around ZIP and Go tricks.
ZIP is a minimal VM, and other machinery such as terms, continuations, and trampolines were implemented as simple Go objects.
Since these were ordinary Go objects, we could use them in foreign predicates.

In contrast, v2 is based on BinWAM which is a single machine that handles Prolog execution and hides the machinery from Go.
I need to define a new API for foreign predicates.

I want writing foreign predicates to feel natural to Go developers, too.


This can be achieved by just two new concepts exposed to Go developers: `Activation` and `Outcome`.

An `Activation` is a scoped interface to the Prolog engine, available only while the goal is active.
It converts between Prolog and Go values and constructs `Outcome` values, as explained below.

An `Outcome` tells the Prolog engine how to proceed with a foreign predicate.
It can be as simple as `a.Succeed()` or `a.Fail()`.
If something goes wrong, it can be `a.Error(err)`.
For multiple answers, it can be `a.Nondet(seq)`, where `seq` is `iter.Seq[prolog.Outcome]`.

For a nondeterministic `Outcome`, the Prolog engine consumes the sequence one result at a time,
requesting the next result when backtracking requires it.
While the foreign predicate is suspended, the `Activation` keeps track of its terms, even if the Prolog GC relocates them.

```go
if err := p.Register("regexp", func(ctx context.Context, a prolog.Activation, pattern, text, match prolog.Term) prolog.Outcome {
	p, err := a.String(pattern)
	if err != nil {
		return a.Error(err)
	}
	
	t, err := a.String(text)
	if err != nil {
		return a.Error(err)
	}

	 // An output argument can be a variable. Otherwise, check its type for ISO-style error handling.
	if !a.Variable(match) {
		if _, err := a.String(match); err != nil {
			return a.Error(err)
		}
	}

	r, err := regexp.Compile(p)
	if err != nil {
		return a.Error(err)
	}

	return a.Nondet(func(yield func(prolog.Outcome) bool) {
		for _, m := range r.FindAllString(t, -1) {
			s, err := a.NewString(m)
			if err != nil {
				_ = yield(a.Error(err))
				return
			}

			if !yield(a.Unification(match, s)) {
				return
			}
		}
	})
}); err != nil {
	panic(err)
}
```

This way, Go developers can describe what their predicates do without knowing how Prolog executes them internally.
