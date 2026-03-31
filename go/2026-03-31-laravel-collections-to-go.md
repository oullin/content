---
title: "Laravel Collections to Go"
excerpt: "Go slices aren't broken. We're not here to complain about them. The issue is ergonomics at scale — what happens when you write the same patterns constantly across a large codebase?"
slug: "2026-03-31-laravel-collections-to-go"
published_at: 2026-03-31
author: "gocanto"
categories: "go"
tags: ["engineering", "go", "maps", "slice", "collection"]
---

![go-maps](https://github.com/user-attachments/assets/e29724b9-c2e2-4378-9d95-e2449e535984)

# We Ported Laravel Collections to Go — Here's Why and How

Go's standard library gives you slices and maps. They're fast, predictable, and honest. But working with them day-to-day means writing the same boilerplate loops over and over — filter this slice, transform that one, take the first five, check if any match a condition. It's not hard, it's just noise. Code that says *how* it works instead of *what* it does.

[Laravel](https://laravel.com/) developers have had a better option for years: the Collections API. A fluent, chainable interface that turns data pipelines into something you can read like a sentence. We wanted that in Go — type-safe, lazy-capable, and built for the way Go actually works today.

So we built it: [github.com/oullin/collection](https://github.com/oullin/collection).


## The Problem With Go Slices (It's Not What You Think)

Go slices aren't broken. We're not here to complain about them. The issue is ergonomics at scale — what happens when you write the same patterns constantly across a large codebase?

Consider this: filter a slice to keep only even numbers, then take the first three, then format each one as a string. In vanilla Go:

```go
var evens []int
for _, n := range numbers {
    if n%2 == 0 {
        evens = append(evens, n)
    }
    if len(evens) == 3 {
        break
    }
}

var result []string
for _, n := range evens {
    result = append(result, fmt.Sprintf("Number: %d", n))
}
```

It works. It's clear to anyone who knows Go. But it's five lines of structural noise to say "give me the first three even numbers, formatted." Every time you write something like this, you're narrating the mechanics instead of expressing the intent.

Now the same thing with our library:

```go
numbers := collection.Collect([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10})

even := numbers.
    Filter(func(n int, _ int) bool { return n%2 == 0 }).
    Take(3)

result := collection.Map(even, func(n int, _ int) string {
    return fmt.Sprintf("Number: %d", n)
})

fmt.Println(result.All()) // ["Number: 2", "Number: 4", "Number: 6"]
```

That reads like what it does. The intent is in front; the mechanics are behind the curtain.


## How We Built It: The Design Decisions

### Go Generics, Not `interface{}`

Before Go 1.18, any attempt at a collections library in Go required either code generation or the `interface{}` escape hatch — and the latter meant losing type safety at compile time and paying for type assertions at runtime. Neither option felt right for production code in regulated systems.

Go 1.18 changed that with generics. Our library is built entirely on them. `Collection[T]` is a proper generic type. The `Filter`, `Take`, `Each`, and `Reject` methods all preserve the type parameter. You get compile-time guarantees with zero casting.

```go
// This is statically typed. No runtime panics from type mismatches.
users := collection.Collect([]User{...})

active := users.Filter(func(u User, _ int) bool {
    return u.Active
})
```

### The Map Function Is a Top-Level Function (On Purpose)

One quirk worth understanding: in Go, methods on a generic type cannot introduce *new* type parameters. That means `Collection[T].Map()` can't return a `Collection[U]` — it's a language-level constraint.

So we made type-transforming operations top-level package functions instead:

```go
// Map is a package-level function, not a method.
// This is the correct Go design — not a limitation we worked around.

result := collection.Map(users, func(u User, _ int) string {
    return u.Name
})

// result is Collection[string]
```

This is the honest Go way to handle it. We didn't paper over the constraint — we designed around it.

### Immutable by Default

Every transformation returns a new collection. The original is never modified. This matters a lot in concurrent code and in systems where you need to reason clearly about state.

```go
original := collection.Collect([]int{1, 2, 3, 4, 5})
filtered := original.Filter(func(n int, _ int) bool { return n > 2 })

fmt.Println(original.All()) // [1, 2, 3, 4, 5] — untouched
fmt.Println(filtered.All()) // [3, 4, 5]
```

### Lazy Evaluation for Large Datasets

For cases where you're dealing with large or potentially infinite sequences, loading everything into memory upfront is the wrong move. Our `lazy` package uses Go's `iter.Seq` — the standard iterator type introduced in Go 1.23 — to process items one at a time, only when needed.

```go
// Nothing is evaluated until you start consuming the sequence.
seq := lazy.From(hugeSlice).
    Filter(isRelevant).
    Take(100)

for item := range seq.Seq() {
    process(item)
}
```

This means you can express the same fluent pipeline semantics over a 10-item slice and a 10-million-row dataset, with the right execution model for each.


## The Package Ecosystem

We structured the library into five focused packages rather than a single monolith. You can use what you need.

**`collection`** — The core package. Wraps a `[]T` in a `Collection[T]` struct with the full fluent API: `Filter`, `Reject`, `Take`, `Skip`, `Each`, `First`, `Last`, `Contains`, `Unique`, `Flatten`, `Chunk`, `Reverse`, and more.

**`lazy`** — Deferred sequences backed by `iter.Seq`. Use this when you're processing streams, piping from a database cursor, or working with data too large to hold in memory at once.

**`collectible`** — A fluent key-value collection: `collectible.Collection[K, V]`. Ordered, iterable, with the same pipeline-style API. Useful when your data has meaningful keys, and you want to avoid the chaos of `map[string]any`.

**`arr`** — Standalone utility functions for raw `[]T` slices. When you don't need the full collection wrapper — just a quick `Sort`, `Flatten`, `Unique`, or `Contains` on a slice you already have.

**`kv`** — Dot-notation access for `map[string]any`. If you're working with decoded JSON blobs or dynamic configuration maps, this lets you write `kv.Get(data, "user.profile.name")` instead of nesting type assertions three levels deep.


## Getting Started

```bash
go get github.com/gocanto/collection
```

A minimal example to show the shape of the API:

```go
package main

import (
    "fmt"
    "github.com/gocanto/collection/collection"
)

func main() {
    numbers := collection.Collect([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10})

    result := numbers.
        Filter(func(n int, _ int) bool { return n%2 == 0 }).
        Reject(func(n int, _ int) bool { return n > 8 }).
        Take(3)

    fmt.Println(result.All()) // [2, 4, 6]
    fmt.Println(result.Count()) // 3
    fmt.Println(result.First()) // 2
}
```

For the full API reference, lazy sequences, key-value collections, and the `arr`/`kv` utility docs, see the [documentation on GitHub](https://github.com/oullin/collection/tree/main/docs).


## Why We're Open Sourcing This

We use this library in our own client work. When we write data-heavy Go services — processing financial records, transforming event streams, working through large query result sets — this is what we reach for instead of writing the same loop patterns by hand.

We're open-sourcing it because the Go ecosystem deserves a production-quality, generics-first collections library. The code is MIT licensed, fully tested, and built to the same standards we apply to regulated systems.

If you're writing Go, and you find yourself typing `for _, item := range` more than you'd like, give it a try.


*The source is at [github.com/oullin/collection](https://github.com/oullin/collection). Questions and contributions are welcome.*
