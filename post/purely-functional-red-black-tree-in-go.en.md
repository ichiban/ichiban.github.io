+++
date = '2025-12-31T16:03:01+09:00'
title = 'Purely Functional Red-Black Tree in Go'
tags = ['Go', 'data structure']
+++

I released a Go library, [`github.com/ichiban/rbtree`](https://github.com/ichiban/rbtree), which implements a purely functional red-black tree.
In this article, I’ll explain what that means and how the implementation works.

# Recap of Map in Go

A **map** in Go (more generally, a hash table) stores key–value pairs and supports insertion, update, and deletion.

[Since Go 1.24, maps are implemented using Swiss Tables](https://go.dev/blog/swisstable), a variant of open-addressing hash tables.

```go
m := map[string]int{}
m["a"] = 1     // map[a:1]
m["b"] = 2     // map[a:1 b:2]
m["c"] = 3     // map[a:1 b:2 c:3]
m["a"] = 4     // map[a:4 b:2 c:3]
delete(m, "b") // map[a:4 c:3]
````

An obvious but important property of Go maps is that they are mutable.
After an operation, you can’t access prior versions unless you explicitly saved them.
To preserve a version, you must copy it into a new map.

```go
snapshot := make(map[string]int, len(m))
for k, v := range m {
	snapshot[k] = v
}
// Or: snapshot := maps.Clone(m)
```

However, not all key–value data structures work this way.

# Purely Functional Red-Black Tree

In Chris Okasaki’s *Purely Functional Data Structures*, he describes a purely functional variant of [red-black trees](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree) that supports insertion, update, and deletion for ordered keys.

The key property is persistence: updates are non-destructive, so you can keep references to any previous version.

The diagram below shows how the same operations affect a red-black tree. Each node has a color (red or black), but the diagram omits it. After an update, existing nodes remain unchanged. So if you keep the old root, you can still access the older version of the tree.

```goat
Insert a:1.

     | 
     v
    a:1


Insert b:2.

              |    
              v
    a:1      a:1   
               \   
                v
                b:2


Insert c:3.
                           |
                           v
    a:1      a:1          b:2
               \          / \
                v        v   v
                b:2     a:1  c:3


Update a:4.
                                        |        
                                        v
    a:1      a:1          b:2          b:2   
               \          / \          / \   
                v        v   v        v   |
                b:2     a:1  c:3     a:4  |
                              ^           |
                              '-----------'

Delete b:2.
                                                    |
                                                    v
    a:1      a:1          b:2          b:2         a:4
               \          / \          / \           \
                v        v   v        v   |           v
                b:2     a:1  c:3     a:4  |          c:3
                              ^           |
                              '-----------'
```

If we bring this data structure into Go, we can build a map-like type that lets us access previous versions without explicitly saving full copies.

# Bringing It to Go

Since Go 1.18 introduced generics, we can define a new data structure `rbtree.Map[K, V]`, where `K` is `cmp.Ordered` and `V` is `any`.

You might consider defining `Node` with pointers to child nodes, but that’s undesirable here: with many nodes, that would create a large pointer graph, increasing GC work and reducing throughput.

```go
// BAD: This increases GC overhead due to a large pointer graph.
type Node[K cmp.Ordered, V any] struct {
	Color       Color
	Left, Right *Node
	KeyValue[K, V]
}
```

Instead, we allocate nodes in a `[]Node[K, V]` slice and refer to children by index.

```go
// Node is a tree node. []Node works as a memory region for a tree.
// Instead of *Node, we refer to Left and Right children by index.
type Node[K cmp.Ordered, V any] struct {
	Color       Color
	Left, Right int
	KeyValue[K, V]
}
```

Then `Map[K, V]` becomes just two fields: a pointer to the node slice and an index of the root node.

```go
// Map is a map backed by a binary search tree.
type Map[K cmp.Ordered, V any] struct {
	Nodes *[]Node[K, V]
	Root  int
}
```

Each update allocates a small number of new nodes and returns a new root index.

```goat
    Map[K, V]                  []Node[K, V]

      .---.                    .---+-----+-----+-----+---+-----.
      | *-+------------------->|  #|color|left |right|key|value|
      +---+                    +---+-----+-----+-----+---+-----+
      | 16|------.             |  0|red  |     |     |a  |    1|
      '---'      | Insert a:1. +---+-----+-----+-----+---+-----+
                 +------------>|  1|black|     |     |a  |    1|
                 |             +---+-----+-----+-----+---+-----+
                 |             |  2|red  |     |     |b  |    2|
                 | Insert b:2. +---+-----+-----+-----+---+-----+
                 +------------>|  3|black|     |    2|a  |    1|
                 |             +---+-----+-----+-----+---+-----+
                 |             |  4|red  |     |     |c  |    3|
                 |             +---+-----+-----+-----+---+-----+
                 |             |  5|red  |     |    4|b  |    2|
                 |             +---+-----+-----+-----+---+-----+
                 |             |  6|black|     |    5|a  |    1|
                 |             +---+-----+-----+-----+---+-----+
                 |             |  7|black|     |     |a  |    1|
                 |             +---+-----+-----+-----+---+-----+
                 |             |  8|black|     |     |c  |    3|
                 |             +---+-----+-----+-----+---+-----+
                 |             |  9|red  |    7|    8|b  |    2|
                 |             +---+-----+-----+-----+---+-----+
                 |             | 10|black|    7|    8|b  |    2|
                 |             +---+-----+-----+-----+---+-----+
                 |             | 11|black|     |     |a  |    4|
                 | Update a:4. +---+-----+-----+-----+---+-----+
                 +------------>| 12|black|   11|    8|b  |    2|
                 |             +---+-----+-----+-----+---+-----+
                 |             | 13|red  |     |     |a  |    4|
                 |             +---+-----+-----+-----+---+-----+
                 |             | 14|red  |     |     |c  |    3|
                 |             +---+-----+-----+-----+---+-----+
                 |             | 15|red  |     |   14|a  |    4|
                 | Delete b:2  +---+-----+-----+-----+---+-----+
                 '------------>| 16|black|     |   14|a  |    4|
                               +---+-----+-----+-----+---+-----+
```

To snapshot the current state, a shallow copy is sufficient.

```go
snapshot := m
```

