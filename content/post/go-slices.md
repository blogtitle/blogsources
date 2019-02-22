---
title: "Go: slices gotchas"
date: 2019-02-22T21:12:56+01:00
categories: ["Go", "Gotchas"]
tags: ["Golang", "Go"]
authors: ["Rob"]
---

# Preface
One of the features that I love the most about Go is the fact that it is **unsurprising**. One could even say that it is _boring_ in some sense.
This is a **good** trait of a programming language. When you code you should focus on the problem at hand, and not on [what your language is doing that you don't want](https://twitter.com/chordbug/status/1092824183124488192?s=19).

This article talks about one of the most "surprising" features of Go for newcomers: slices.

# Basic usage
If you know how to use Go slices, please skip to the next section.

You can declare a slice:
```go
var a []int
```
Slices have literals:
```go
a := []int{1, 2}
```
Slices are collections of variable length. Unlike arrays they can be grown and sub-sliced at will.

Arrays:
```go
// Array of zeroes, size 4
var a [4]int
// Array literal of zeroes, size 3
b := [...]int{2: 0}
// Array literal of zeroes, size 2
c := [...]int{0, 0}
// Both are invalid: [4]int, [3]int and [2]int are different types
a = b
c = b
```
Slices:
```go
// Slice of zeroes, size 4
a := make([]int, 4)
// Slice literal of zeroes, size 3
b := []int{2: 0}
// Slice literal of zeroes, size 2
c := []int{0, 0}
// Allowed: []int and []int are the same type
a = b
c = a
```
Moreover, slices can be sub-sliced:
```go
a := []int{0, 1, 2, 3, 4}
b := a[1:3] /* [1, 2]          */
c := a[3:]  /* [3, 4]          */
d := a[:2]  /* [0, 1]          */
e := a[:]   /* [0, 1, 2, 3, 4] */
```
And extended:
```go
a := []int{1, 2}
b := append(a, a...) /* [1, 2, 1, 2] */
a = append(a, 3, 4)  /* [1, 2, 3, 4] */
```
This generally makes slices the data structure of choice for all use cases.
# So, what's wrong?
Slices are nothing more than a `struct` carrying 3 pieces of information:
```go
type slice struct {
	// Size of used space in data
	len  int
	// Size of data
	cap  int
	// Underlying array
	data *[...]Type
}
```
When a slice of a slice is taken, `cap`, `len` and `data` might change, but **the underlying array is not re-allocated, nor copied over**.

This creates some weird behaviors.

## Ghost updates: part 1
```go
a := []int{1, 2}
b := a[:1]     /* [1]     */
b[0] = 42      /* [42]    */
fmt.Println(a) /* [42, 2] */
```
This kind of trickery is mostly expected by gophers, usually because some core interfaces of the language rely on the fact that slices' underlying data is passed by reference.
For example `io.Reader` has the same type signature of `io.Writer`, which might be rather surprising for newcomers:
```go
type Reader interface {
	// Read overwrites data in p
	Read(p []byte) (n int, err error)
}
type Writer interface {
	// Write reads data from p
	Write(p []byte) (n int, err error)
}
```

## Ghost updates: part 2
This is where it gets trickier
```go
a := []int{1, 2, 3, 4}
b := a[:2] /* [1, 2] */
c := a[2:] /* [3, 4] */
b = append(b, 5)
fmt.Println(a) /* [1 2 5 4] */
fmt.Println(b) /* [1 2 5]   */
fmt.Println(c) /* [5 4]     */
```
When data gets appended to `b`, the underlying array has enough capacity to hold two more elements, so append will not re-allocate. This means that appending to `b` might change `c`.

## Ghost updates: part 3
```go
a := []int{0}     /* [0]          */
a = append(a, 0)  /* [0, 0]       */
b := a[:]         /* [0, 0]       */
a = append(a, 2)  /* [0, 0, 2]    */
b = append(b, 1)  /* [0, 0, 1]    */
fmt.Println(a[2]) /* 2 <- Legit   */

// Identical code, just starting with bigger slice:
c := []int{0, 0}  /* [0, 0]       */
c = append(c, 0)  /* [0, 0, 0]    */
d := c[:]         /* [0, 0, 0]    */
c = append(c, 2)  /* [0, 0, 0, 2] */
d = append(d, 1)  /* [0, 0, 0, 1] */
fmt.Println(c[3]) /* 1 <- ??      */
```
The reason for this odd behavior is that when a slice becomes bigger than a certain threshold, go stops linear growth and starts allocating slices that are double the size of the previous one. This **depends on the size of the slice type**.

To go more in details:

* The first `append` to `a` copied the previous zero into a slice of `cap == 2`, and wrote a `0` in `a[1]`
* A slice of `a` was taken, `len(b) == cap(b) == 2`
* The second `append` to `a` copied the previous zeroes into a slice of `cap == 4`, and wrote `2` in `a[2]`
* At this point `b` was still with `cap == 2`, so the `append` on `b` allocated a new backing array.

The same procedure, starting from an original `cap` of 2 yields a different result because when we take a slice of `c` it has already grown to `cap == 4`.

> Trivia: since this behavior depends on the size of the underlying type, `[]struct{}{}` will always be grown by the exact amount of elements appended.

## How do I fix this?
If you pass around slices that are never appended to, you are safe. Just keep in mind that everyone shares a "view" on the same memory area. The same holds true if the functions you call do not keep a reference to your slices after they return.

If instead you plan to pass around slices that someone might append data to, and you also plan to grow the original one, you might want to consider limiting the capacity on the data you share.

```go
a := append([]int{}, 0, 1, 2, 3)
// The following call might be dangerous if
// `potentialSliceGrower` keeps a reference to `a`.
potentialSliceGrower(a)
// This is safer: take a slice of the exact size.
// Appending will cause copying.
potentialSliceGrower(a[:4:4])
```
This unusual 3-index syntax takes a slice of `a` that starts at index `0`, ends at index `4` and has `cap = 4`.

Please use this **only if needed**, but don't forget to do it when necessary.

# Want to know more?
There is an official go blog [post](https://blog.golang.org/go-slices-usage-and-internals) with slices internals and [another gotcha at the end](https://blog.golang.org/go-slices-usage-and-internals#TOC_6.).
