---
title: "Go less used features"
date: 2019-02-06T19:39:41+01:00
categories: ["Go"]
tags: ["golang"]
authors: ["Rob"]
draft: true
---

# Preface
I've been using Go for a while now, and I really like to dig deep in internals of tools that I use. This led me to discover a lot of go hidden and peculiar features that I very rarely use.

Here is a collection of both well known and unknown go quirks that I almost never use

# Put methods on EVERYTHING
Go methods and interfaces are very simple: there is no overload, no covariance or contravariance, nothing too fancy. This comes with some perks othern than a very readable and fast to compile source. For one, interface matching is very simple and implicit, and you can put methods on almost anything you can name.

### On native types
```go
type myErr string

func (e myErr) Error() string {
	return string(e)
}

// This is valid: myErr implements the error interface.
var err error = myErr("Something went wrong")
```
This is exactly what [`errors.New`](TODO) does.

### On funcs
Methods can be put on functions, so that you can easily implement single-method interfaces with just the code if there is no need for a state:
```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

// ServeHTTP implements the http.Handler interface for HandlerFunc
func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request){
	f(w,r)
}
```
This is the exact implementation of [`http.HandlerFunc`](TODO).

### On interfaces
If you ever find yourself in need of putting methods on interfaces you can **embed interfaces in structs**:
```go
type myReader struct{ io.Reader }

func (r myReader) ReadByte() (byte, error) {
	b := make([]byte, 1)
	// Use r as an io.Reader
	_, err := io.ReadAtLeast(r, b, 1)
	if err != nil {
		return 0, err
	}
	return b[0], nil
}

func main() {
	r := myReader{strings.NewReader("B")}
	b, _ := r.ReadByte()
	fmt.Printf("%x", b) // 42
}
```
In this case `myReader` is both an `io.Reader` (Read gets promoted by embedding) and has the `ReadByte` method.

# Take a method as a func
To be honest I used this feature only once and I'm still unsure if it was a correct use case. In Go methods are not special `func`, they are almost just some syntactic sugar to say that the first parameter is of a certain type.

For example in the following code:
```go
type A string
func (a A) B() string {
	return string(a)
}
```
`B` is just a `func(a A) string`. In some rare cases it might come in handy to "detach" a method from a type and pass it around as a function.
Considering type `A` above:
```go
var detached func(a A) string = A.B
a := A("foo")
// This prints "foo"
println(detached(a))
```

# Three indices slices
Go has slices, and they have almost all features that you'd like them to have, plus and minus one. The feature they lack is negative indexing, the unexpected feature they have is three-indices slice operator:
```go
a := []int{2, 3, 4}
b := a[0:2:2]
```
b is a slice of a of length 2 and capacity 2. This makes it so that appending to b will allocate a new slice and not overwrite elements of a. If you are interested I wrote a dedicated post on [Go slices weirdness](TODO).

# Three parameters make
`make` is a builtin `func` that allocates memory for new native types. It usually takes one or two parameters: the first is the type and the second (optional) is the size. One exceptional case is when slices are being allocated, in that case you can go up to 3 by also specifying capacity.

```go
a := []int{1, 2, 3}
// make(Type, len == cap)
b := make([]int, 3)
// Correct use of copy
copy(b, a)

c := make([]int, 3)
for _, v := range a {
	// This is wrong as it appends after element 2
	c = append(c, v)
}
fmt.Println(c) // [0 0 0 1 2 3]

// Tell Go that slice should have room for
// three elements, but currently has 0.
// make(Type, len, cap)
d := make([]int, 0, 3)
for _, v := range a {
	d = append(d, v)
}
fmt.Println(d) // [1 2 3]
```
I would still strongly advise against pre-allocating slices, grow them dynamically from `nil`:
```go
a := []int{1, 2, 3}
var b []int
for _, v := range a {
	// Append grows nil slices for you
	b = append(b, v)
}
```

# Multiple assignments
Multiple assignments are very common in function calls:
```go
value, err := function(input)
```

This comes in very handy if you want to swapt two variables, for example to implement [`sort.Interface`](TODO)
```go
// There is no need for a temporary swap variable
a, b = b, a
```

# fallthrough
The `switch` statement has a default `break` behavior at the end of every `case`. Unlike most languages control flow will not "fall" into the following `case`. If you find yourself in the very uncommon situation in which you need to execute the next case, you can have `fallthrough` as the **last statement of a case**.
```go
var greater10, greater100 bool
switch {
case x > 100:
	greater100 = true
	fallthrough
case x > 10:
	greater10 = true
}
```
In the even more remote situation in which you need to fallthrough from a case to the following one, but not as the last statement, please refactor your code.

If you still want to do it because you want to trick your future self into a weeks long debug session, you can use a `goto`:
```go
switch {
case IHateMyself:
	if IHateMeEvenMore {
		goto Fallthrough
	}
	doSomething()
Fallthrough:
	fallthrough
case WhyDidIDoThis:
	doSomethingElse()
}
```

# complex
Go has builtin support for [complex numbers](TODO). I haven't **properly** used them yet, but they come in handy if you need to represent a bidimentional vector and don't have time or will to re-implement basic operations.

# goto
Go has a low-power `goto` statement: you cannot do too many things with gotos. It is invalid to change scope in a way that would give visibility to different variables that are not live from the call site. TODO use cases in the stdlib

# else

# recover (add that it only recovers panics down the stack)

# recursive types (the stateFn type in text/template)
