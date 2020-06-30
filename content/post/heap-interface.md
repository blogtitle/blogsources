---
title: "Go Heap Interface"
date: 2019-10-12T18:17:01+02:00
categories: ["Go", "Software engineering"]
tags: ["golang", "interfaces"]
authors: ["Rob"]
draft: true
---

### Discovering the heap

As of recently I had to use the `container/heap` package. For those who are unfamiliar with it, it works similarly to the `sort` package: it provides functions that accept an `heap.Interface` type and established heap invariants on the underlying collection.

An heap is just a special kind of sorted collection, so I expected the interface to be identical to `sort.Interface`, but instead it adds two more methods: `Push(interface{})` and `Pop() interface{}`.

```go
// package heap

type Interface interface {
        sort.Interface
        Push(x interface{}) // add x as element Len()
        Pop() interface{}   // remove and return element Len() - 1.
}
```

Let's take a look at `sort.Interface`:

```go
// package sort

type Interface interface {
        // Len is the number of elements in the collection.
        Len() int
        // Less reports whether the element with
        // index i should sort before the element with index j.
        Less(i, j int) bool
        // Swap swaps the elements with indexes i and j.
        Swap(i, j int)
}
```

These three methods provide primitives to the `sort` package to reorder the underlying collection (usually one or multiple slices).

The `heap` package, instead, asks the user for two more methods that modify the length of the container.

I don't like this because I wanted to:

* stay in control of my slice
* not use `interface{}` (`sort.Sort` doesn't)
* be able to implement Push, Pop and Peek with just a few lines and primitives offered by `heap`.

In addition to that, `heap` is quite confusing and clunky: `Push` and `Pop` are not supposed to be called by the user, which has to call `heap.Push` and `heap.Pop`. `Peek` just does not exist and will likely not exist as the next item to be removed is in position 0 of the underlying collection (see [#17510](https://github.com/golang/go/issues/17510).

This felt like a quite arbitrary mix of primitives and higher level functions. After it bit me a couple of times I decided to dig into the sources.

The plumbing inside the whole package is just two functions called `up` and `down`, which are the sole responsible of keeping the heap in order, and they don't need to use `interface{}` or resize the underlying slice.

### Fixing the heap

I thus decided to revise the package a bit:

* I decided to just accept `sort.Interface`: it should be enough to re-establish heap invariants
* Peek, Pop and Push would need to be implemented by the user, but they would even with the previous implementation
* There would be no name clash (e.g. `heap.Interface.Pop` vs `heap.Pop`)
* No use of `interface{}`

Heap currently exposes

```go
func Fix(h Interface, i int)
func Init(h Interface)
func Pop(h Interface) interface{}
func Push(h Interface, x interface{})
func Remove(h Interface, i int) interface{}
```

But I think that `Fix` and `Init` are more than enough, since it is probably what users are after.

If `s` is the underlying slice and it implements `sort.Interface`, `Pop` can be implemented as
```go
func (s *myheap) Pop() Type {
  // Save removed item
  popped := s[0]
  // Move item 0 to the end of the heap
  s.Swap(0, s.Len())
  // Crop the slice
  *s = s[:len(s)-1]
  // Re-establish heap invariants
  heap.Fix(s, 0)
  return popped
}
```

With the current implementation this same operation requires 5 lines (you can find it in the package examples).


