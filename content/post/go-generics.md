---
title: "Go Generics"
date: 2021-05-07T05:54:36+02:00
categories: ["Go"]
tags: ["golang"]
authors: ["Rob"]
---

# Preface
Go doesn't have generics yet, but there is [a proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md) that looks promising, and this post refers to that.

[Here](https://go2goplay.golang.org/) is a version of the playground you can use to try this proposal out if you want to experiment without installing any tools.

# Functions
### Converting a slice into a channel.
This is not necessarily the most useful function but it's a good and simple example:
```go
func chanFromSlice[T any](in []T) chan T {
  c := make(chan T, len(in))
  for _, v := range in {
    c <- v
  }
  return c
}
```
Here you can see that this looks and feels like Go, with the only difference of the `[T any]` bit.
This instructs the compiler to read the rest of the function in a "generic" way, so every time `T` appears it knows it refers to the same type.
The built-in constraint `any` means that the type is not constrained and can be anything.

Using this function looks like this:
```go
c := chanFromSlice([]int{1,2,3})
```
Note that we don't have to tell the compiler we need the `int` version of this function.
When this code is compiled it will automatically realize that we are passing an `[]int`, so it will understand that it has to compile `chanFromSlice` with `T = int`.

We will see that this inference can't always be applied, but in most use cases **users of generics won't even realize they're using generic functions**.

### A slice allocator
Let's say we want to allocate a slice that can accommodate one item for every item in another slice, but of a different type:
```go
func allocSlice[T, U any](other []T) []U {
  return make([]U, 0, len(other))
}
```
If we try to use this function, the compiler will complain:
```go
x := []int{1, 6, 3}
// ERROR: cannot infer U ([int <nil>])
y := allocSlice(x)
```
So in this case we **have to** pass the type we want the new slice to be of:
```go
x := []int{1, 6, 3}
y := allocSlice[int, string](x)
```

### Keys from a map
I think we've all written this kind of code at least once, so I'm quite happy this is now possible to express:
```go
func keys[K comparable, V any](m map[K]V) []K {
  keys := make([]K, 0, len(m))
  for k := range m {
    keys = append(keys, k)
  }
  return keys
}

func main() {
  m := map[string]bool{
    "foo": true,
    "bar": false,
  }
  fmt.Println(keys(m)) // [foo bar]
}
```
Here you can see I used `comparable`, another builtin constraint that encompasses all types that have the `==` operator defined on them.

### Sorting a slice
Currently the standard library provides us with a very helpful [`sort.Slice`](https://golang.org/pkg/sort/#Slice) function.
The minor issue with this is that it takes anything, even a `struct{}` as first argument, and this means that a mistake that could be caught at compile-time will only surface at runtime:

```go
// This compiles but panics
sort.Slice("not a slice", func(i, j int) bool{return true})
```

Let's make it safe:
```go
func sortSlice[T any](x []T, less func(i, j int) bool) {
  sort.Slice(x, less)
}
```
This can be used as simply as this:
```go
x := []int{1, 6, 3}
sortSlice(x, func(i, j int) bool { return x[i] < x[j] })
fmt.Println(x) // [1 3 6]
```
Once again the compiler will detect the type, no need to pass it, and `sortSlice("not a slice", ...)` will not compile.

### Sorting any slice of comparable types
We have `sort.Ints` and `sort.Strings` in the standard library, but what if we want to sort a slice of `float64`?
What about `int32`?

Let's make a helper for this:
```go
// ordered is a type constraint that matches any type that supports the < operator.
type ordered interface {
  type int, int8, int16, int32, int64, uint, uint8,
    uint16, uint32, uint64, uintptr, float32, float64, string
}

func sortSlice[T ordered](s []T) {
  sort.Slice(s, func(i, j int) bool {
    return s[i] < s[j]
  })
}
```

As you can see this time we didn't use a builtin constraint like `any` or `comparable` but we created our own.

# Types
### A type-safe pool
The standard library offers a [`sync.Pool`](https://golang.org/pkg/sync/#Pool) type to relieve the garbage collector from the stress of having to allocate and release the same type over and over again.
This is normally used for buffers, for example in programs that have to copy a lot of data many times per second.

Here is the Pool API:
```go
type Pool struct {
    // New optionally specifies a function to generate
    // a value when Get would otherwise return nil.
    New func() interface{}
}
func (p *Pool) Get() interface{};
func (p *Pool) Put(x interface{});
```

Seemingly simple, apparently straightforward, except when you're using it and accidentally put a `*[]byte` in it instead of a `[]byte` and the next time you draw from the pool your program crashes.

Let's make it safe:
```go
type Pool[T any] struct {
  p *sync.Pool
}

func NewPool[T any](new func() T) Pool[T] {
  return Pool[T]{
    p: &sync.Pool{
      New: func() interface{} {
        return new()
      },
    },
  }
}

func (p Pool[T]) Get() T {
  return p.p.Get().(T)
}

func (p Pool[T]) Put(t T) {
  p.p.Put(t)
}
```

Something very similar could be done with the `sync.Map` type to make sure we don't mistakenly pass the wrong type as key or value.

Similarly, one could implement a `Set[T comparable]` type that uses a `map[T]struct{}` underneath.

### A locked type
If we want to make a type that can protect something from concurrent access, we can create a small container:
```go
type Locked[T any] struct {
  mu sync.Mutex
  t  T
}

func NewLocked[T any](t T) *Locked[T] {
  return &Locked[T]{
    t: t,
  }
}

func (l *Locked[T]) Acquire() T {
  l.mu.Lock()
  return l.t
}

func (l *Locked[T]) Release(t T) {
  l.t = t
  l.mu.Unlock()
}
```

And we can use it with deferred or sequential calls:
```go
l := NewLocked(3)

func() {
  i := l.Acquire()
  defer func() {
    l.Release(i)
  }()
  i = 100
}()

i := l.Acquire()
fmt.Println(i) // 100
l.Release(i)
```

# Constraints
### Parametric constraints
You can have a constraint take a type parameter:

```go
type getter[T any] interface {
  Get() T
}

func getInt[G getter[int]](g G) int {
  return g.Get()
}
```

### Type lists
As we already saw in the sorting example, constraints can be list of types:
```go
type ordered interface {
  type int, int8, int16, int32, int64, uint, uint8,
    uint16, uint32, uint64, uintptr, float32, float64, string
}
```

### Mixed
You can define a constraints to be a type list and have methods:
```go
type FloatSliceStringer interface {
  type []float64, []float32
  fmt.Stringer
}

type myFloat []float64

func (m myFloat) String() string {
  return fmt.Sprint(m)
}

func check[T FloatSliceStringer](t T) {}

func main() {
  var f myFloat
  check(f) // Compiles
}
```

### Self referring
You can even write a constraint that refers to the same type.
This can't be done in a type declaration, but it can be done in a function signature.
Let's say we want to write a function that accepts all types that can be cloned and clones them:

```go
func clone[Cloneable interface{ Clone() Cloneable }](c Cloneable) Cloneable {
  // The signature guarantees the type returned by Clone is the same of c
  return c.Clone()
}
```

### Type parameters in type lists
If you want to express a constraint that accepts all slices, even named types, you can:
```go
type SliceConstraint[T any] interface {
  type []T
}
```
Note that this is different from just having a function accept a `[]T` since there are types that can be defined as
```go
type MySlice []int
```
and you might want to treat them as `MySlice`, not just as `[]int`.

Alternatively you can use the approximation of the type [as proposed here](https://github.com/golang/go/issues/45346).

# Main Limitations
### No type switch
It is currently not possible to specialize code for some specific types of a constraint:

```go
type floats interface {
  type float64, float32
}

func specialized[F floats](f F) {
  // ERROR: f (variable of type parameter type F) is not an interface
  switch x := f.(type) {
  case float64:
  case float32:
  }
}
```
You can assign a type parametric variable to `interface{}` and then perform a type switch, but that will be executed at runtime:
```go
func specialized[F floats](f F) {
  var i interface{} = f
  // Runtime switch
  switch x := i.(type) {
  case float64:
    fmt.Printf("%T(%v)\n", x, x)
  case float32:
    fmt.Printf("%T(%v)\n", x, x)
  case string:
    // The compiler can't tell this is impossible
  }
}

func main() {
  specialized(float64(1)) // float64(1)
  specialized(float32(2)) // float32(2)
}
```

### No generic methods
While we can have generic types, their methods can only take the type parameters and no more:
```go
type Foo[T any] struct {
  t T
}

// ERROR: methods cannot have type parameters
func (f Foo[T]) something[U any]() {}
```

Note that this is very unlikely to change since accepting type parameters on methods would make interface matching odd and surprising.

### A type must always be the same
If you have a function defined as `func f[T any](slice []T)` all elements of `slice` must be of the same type.
If you want to express the concept of "_A slice of elements, each element of an arbitrary type_" you still have to use `func f(slice []interface{})`.
This is working as intended since interfaces and generics aim to solve different problems.

### No casting
There is just no way to express a constraint for two types to be convertible to each other:
```go
// This can't be expressed
func cast[T1, T2 convertible[T1])(t2 T2)T1{
  return T1(t2)
}
```

# Opinions
I actually like this proposal, and I can't wait to use this.
I won't be writing type parametric code every day, but I feel like I'll be using it for stuff like sorting, max and sets quite often.
I plan to be a consumer of generic APIs, but there are very few places where I'll implement them.

I can see how this new feature might be abused, but I am hopeful that the community will self-regulate and we won't end up with a Java-Go hybrid abomination.
