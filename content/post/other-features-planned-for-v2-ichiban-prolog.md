---
title: Other features planned for v2
date: 2026-11-05T00:00:00+09:00
---

In developing `ichiban/prolog` v2, I've been reimplementing the engine using BinWAM and redesigning the APIs for query and foreign predicates.

Beyond the query and foreign predicate APIs, I'm also working on a few other features for v2.

## Pluggable Dynamic Predicate DB

ISO Prolog supports dynamic predicates whose clauses can be added or removed using `asserta/1`, `assertz/1`, and `retract/1`.
In v1, those dynamic clauses were stored in memory.
I've been asked whether dynamic predicates could be persisted or distributed across multiple instances.
By making the DB pluggable, Go developers can implement their own storage backends, including persistent or distributed ones.

```go
// DB stores the clauses of dynamic predicates. Every write happens at a
// revision strictly greater than any [DB.Revision] has returned, so a Select
// under way never sees a later one.
type DB interface {
	// Revision returns the current revision.
	Revision() Revision

	// Select yields the clauses of module/name/arity as of revision -- inserted
	// at or before it, not deleted as of it -- in clause order.
	Select(ctx context.Context, module, name Atom, arity int, revision Revision) iter.Seq2[Record, error]

	// InsertBefore stores payload ahead of the other clauses of
	// module/name/arity. It may keep the slice.
	InsertBefore(ctx context.Context, module, name Atom, arity int, payload []byte) error

	// InsertAfter stores payload after the other clauses of module/name/arity.
	// It may keep the slice.
	InsertAfter(ctx context.Context, module, name Atom, arity int, payload []byte) error

	// Delete removes the clause. Select still yields it at earlier revisions.
	Delete(ctx context.Context, module, name Atom, arity int, id ClauseID) error
}
```

## SICStus-style Module System

Prolog module systems are surprisingly contentious, even among implementations following the SICStus style.
I'm adopting that style for compatibility, though I don't pretend there's a single definitive interpretation of it.

These are the directions I'm exploring for v2, and the details may change as development progresses.

