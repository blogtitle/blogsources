---
title: "Let's Go to Go2"
date: 2021-05-07T09:04:30+02:00
categories: ["Go"]
tags: ["golang", "generics"]
authors: ["Rob"]
---

# Preface

In [my previous post](blogtitle.github.io/go-generics/) I summarized the [latest generics proposal for Go](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md).

In this post I'm gonna use it a bit, and misuse it a lot.

# The problem
Sometimes I find myself writing code to run the same function in parallel with itself with different inputs, for example to process a batch of data.

This is not super complicated code, but it's not exactly straightforward and can sometimes be tricky to get right.

There are many ways to implement this, but they all look more or less like this:
```go
func work(n int) int {
	// Heavy computation.
	time.Sleep(10 * time.Millisecond)
	return n * n
}

func parallelWork(out chan<- int, in <-chan int, nWorkers int) {
	var wg sync.WaitGroup
	wg.Add(nWorkers)
	for i := 0; i < nWorkers; i++ {
		go func() {
			defer wg.Done()
			for i := range in {
				out <- work(i)
			}
		}()
	}

	wg.Wait()
	close(out)
}

func main() {
	out, in := make(chan int), make(chan int)
	input := []int{3, 4, 1, 5, 5, 7, 9}
	go func() {
		for _, i := range input {
			in <- i
		}
		close(in)
	}()
	go parallelWork(out, in, 4)
	var res []int
	for o := range out {
		res = append(res, o)
	}
	fmt.Println(res)
}
```

Note that the bit I usually want to write and that I want the reader and code reviewer to focus on is `work` and the amount of parallelism the rest is more or less boilerplate or noise.

If we remove the noise from above, we are left with something like this:
```
input := []int{3, 4, 1, 5, 5, 7, 9}
func work(n int) int {
  // Heavy computation.
  time.Sleep(10 * time.Millisecond)
  return n * n
}
res := parallel(4, work, input)
fmt.Println(res)
```

The problem is that without generics it is hard to abstract this kind of pattern and create a reusable function for it.

# The solution

Let's try to build a more generic and readable helper for this.

```go
func parallel1[IN, OUT any](in <-chan IN, work func(IN)OUT, parallelism int)(<-chan OUT){
  out := make(chan OUT)
  var wg sync.WaitGroup
  wg.Add(parallelism)
	for i := 0; i < parallelism; i++ {
		go func() {
			defer wg.Done()
			for i := range in {
				out <- work(i)
			}
		}()
	}
  go func(){
	wg.Wait()
	close(out)
  }()
}
```

With this, our code will become a bit less noisy:
```go
in := make(chan int)
input := []int{3, 4, 1, 5, 5, 7, 9}
go func() {
	for _, i := range input {
		in <- i
	}
	close(in)
}()
out := parallel1(in, 4)
var res []int
for o := range out {
	res = append(res, o)
}
fmt.Println(res)
```

Most of the time in a real program the input won't come from a slice and the outputs will also very likely not go into a slice, it will likely be files or sockets.
Moreover most of these functions won't be generic as they'll likely have to deal with real data.

Anyways, to simulate what would be in a real program we can build helpers.

```go
func fromSlice[T any](s []T)(<-chan T){
  c := make(chan T)
	go func() {
		for _, i := range s{
			c <- i
		}
		close(c)
	}()
}

func toSlice[T any](c <-chan T)[]T{
  var s []T
  for i := range c {
    s = append(s, i)
  }
  return s
}
```

Thanks to these now our program looks like this:
```go
in := fromSlice([]int{3, 4, 1, 5, 5, 7, 9})
out := parallel1(in, 4, work)
res := toSlice(out)
fmt.Println(res)
```

At this point I think we moved away most of the noise without losing in readability.

And this is where we should stop. The only things we could add are a context argument to all functions that spawn goroutines, and maybe an error channel just in case we have to parallelize operations that might fail.

But that's it.

That said, I'm not gonna stop here today because this is not production code, but it's a post on my personal blog.

# Let the abuse begin

**Please do not use this code anywhere**.

Let's start by saying that we should add support for errors and context, but in a standardized and modular way.

To do this we define a few simple helpers that we will use from now on:
```go
// Sink is where data and errors go to.
type Sink[T any] struct {
	Vals chan<- T
	Errs chan<- error
}

// Source is where data and previous errors come from.
type Source[T any] struct {
	Vals <-chan T
	Errs <-chan error
}

// NewBufferedPipe creates a pipe connecting a Sink and a Source, with a buffer.
func NewBufferedPipe[T any](bufSize, errBufSize int) (Sink[T], Source[T]) {
	v := make(chan T, bufSize)
	e := make(chan error)
	return Sink[T]{v, e}, Source[T]{v, e}
}

// NewPipe creates an unbuffered pipe.
func NewPipe[T any]() (Sink[T], Source[T]) {
	return NewBufferedPipe[T](0, 0)
}

// RunPipeCtx runs a pipeline in a child of the provided context.
// When run returns the context is cancelled.
func RunPipeCtx(ctx context.Context, max time.Duration, run func(ctx context.Context, abort func())) {
	var innerCtx context.Context
	var cancel func()
	if max == 0 {
		innerCtx, cancel = context.WithCancel(ctx)
	} else {
		innerCtx, cancel = context.WithTimeout(ctx, max)
	}
	defer cancel()
	run(innerCtx, cancel)
}

// RunPipe is a helper to run a pipe in the background context.
func RunPipe(max time.Duration, run func(ctx context.Context, abort func())) {
	RunPipeCtx(context.Background(), max, run)
}
```

With these, we can start working on building blocks to create arbitrarily complicated pipelines with just a few lines of code and an immense cognitive overhead for the reader.

Let's see some examples with the potential data sources:
```go
func FromChan[T any](c <-chan T) Source[T] {
	return Source[T]{c, make(chan error)}
}

func FromVars[T any](vs ...T) Source[T] {
	return FromSlice(vs)
}

func FromShortSlice[T any](slice []T) Source[T] {
	c := make(chan T, len(slice))
	for _, v := range slice {
		c <- v
	}
	close(c)
	return FromChan(c)
}
```

We can do something similar for `io.Reader` if we are, for example, reading lines from a file:
```go
func FromReader(ctx context.Context, r io.Reader, split bufio.SplitFunc) Source[string] {
	s := bufio.NewScanner(r)
	if split != nil {
		s.Split(split)
	}
	snk, src := NewPipe[string]()
	go func() {
		defer close(snk.Vals)
		for s.Scan() {
			select {
			case <-ctx.Done():
				return
			case snk.Vals <- s.Text():
			}
		}
		if err := s.Err(); err != nil {
			send(ctx, snk.Errs, err)
		}
	}()
	return src
}
```

The possibilities for creation operators are unlimited, we could also take a generator function, but let's stop here and take a look at the actual fun stuff: modifying operators.

These are the signatures of the stuff that will manipulate our pipes:

```go
func Map[In, Out any](ctx context.Context, in Source[In],
    transform func(In) (Out, error)) Source[Out]{

func Parallel[In, Out any](ctx context.Context, in Source[In], size int,
    transform Transformation[In, Out]) Source[Out] {

func Filter[T any](ctx context.Context, in Source[T],
    filter func(T) bool) Source[T] {

func Concat[T any](ctx context.Context, ins ...Source[T]) Source[T] {

func Merge[T any](ctx context.Context, ins ...Source[T]) Source[T] {

func Partition[T any](ctx context.Context, in Source[T],
    matcher func(T) bool) (matches Source[T], nonMatches Source[T]) {

func Tap[T any](ctx context.Context, in Source[T],
    tapVals func(T), tapErrs func(error)) Source[T] {

func BufferCount[T any](ctx context.Context, in Source[T], bufSize int) Source[[]T] {

func Pairwise[T any](ctx context.Context, in Source[T]) Source[[2]T] {
```

Think about it for a second.
One day Go code might look like this:
```go
RunPipe(3*time.Second, func(ctx context.Context, _ func()) {
  urls := FromFile(ctx, "user_urls.txt")
  bodies := Parallel(ctx, urls, 10, request)
  parsed := Parallel(ctx, bodies, 3, parseJSON)
  admins := Filter(ctx, parsed, isAdmin)
  entries := Map(ctx, admins, toJSON)
  err := ToFile(ctx, entries, "admins.txt")
	if err != nil {
    // TODO check this err instead.
    return err
	}
})
```
 Wouldn't that just be the dream?

 Short answer: no.

 The operators variety would get so broad reading any code would require an immense amount of knowledge.
 This kind of stuff reminds me of the difficulty of reading complex pipes in rxjs: I have no doubt those make your code compact, but that isn't always the objective.

 When the go2go tool came out I wrote more than a thousand lines of code just of operators, and I was barely getting started.
 I'm not sure that's what our community should go for. If you want to read the code I'll publish it. Please let me know [in this survey](TODO) but promise you'll never use that code.

# The challenge
The challenge will be to provide **some** helpers, so that it doesn't happen that everyone has to re-implement a threadpool every time they need some parallelism, but at the same time it should keep the code easy to read for newcomers.

This is, I think, the biggest challenge we are going to face in the coming years.
As soon as we get generics I'm sure we'll get a plethora of libraries that do exactly what I talked about in this post, and we'll have to carefully decide how hard we want to make our language to read for the sake of sparing a few keystrokes every now and then.
