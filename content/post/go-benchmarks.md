---
title: "Go Advanced Benchmarking"
date: 2019-01-11T22:15:43+01:00
categories: ["Go", "Optimization"]
tags: ["Benchmark","Go build tags", "go:generate"]
authors: ["Rob"]
---
# The story
Sometimes you have to solve a problem that comes in several flavours. Usually complicated problems do not offer a single solution, but there are several solutions that are optimal or terrible depending on which subset of that problem the program will have to solve at runtime.

One example I faced was to analyse some data flowing in some connections that I was proxying.

There are two main ways to extract some information from traffic: you can either **record the entire traffic** to analyse it as soon as it is done, or you can analyse it **while it flows**(with a buffer window) at the cost of slowing it down.

Memory is relatively cheap compared to processing power, so my first solution to the problem was the buffered one.

### First code: the buffer
Buffering the connection is fairly easy: just copy everything I read into a `bytes.Buffer` and when the connection is closed analyse what I read. Simplest way to do it is to wrap the connection with something that makes call to `Read` go through an [`io.TeeReader`](https://golang.org/pkg/io/#TeeReader).

This was fairly simple to do, and handled pretty well situations with frequent low-traffic, short-lived connections.

```go
type bufferedScanner struct {
	net.Conn
	tee    io.Reader
	buffer *bytes.Buffer
}

func NewBufferedScanner(original net.Conn) net.Conn {
	var buffer bytes.Buffer
	return bufferedScanner {
		Conn: original,
		tee: io.TeeReader(c, &buffer),
		buffer: &buffer,
	}
}

func (b bufferedScanner) Read(p []byte) (n int, err error) {
	return b.tee.Read(p)
}

func (b bufferedScanner) Close() error{
	analyse(b.buffer)
	return b.Conn.Close()
}
```

After small optimizations to [efficiently re-use buffers](https://golang.org/pkg/sync/#example_Pool) I was happy with this solution, at least for a while.

### Second code: the scanner
It wasn't long before I realized that long lived connection or bursty ones were handled poorly by this solution, so I wrote some code that would work in a streamed fashion instead of buffering everything.

This solution ended up being more expensive in terms of initial memory (to build scanners and other additional data structures) but it would become more efficient in both memory and computation after some tens of kilobytes sent over the same connection.

A streamed solution turned out to be trickier to implement, but thanks to the [`bufio`](https://golang.org/pkg/bufio/) package it was manageable. The code is just a wrapper around a [`scanner`](https://golang.org/pkg/bufio/#Scanner) with a custom [`SplitFunc`](https://golang.org/pkg/bufio/#SplitFunc).

### The meta-problem
Regardless how good my solutions were, I now had two pieces of code that had pros and cons: for very low-traffic short lived connections the first one was much better, but for more traffic-intensive ones the second was the only one viable.

I had two possibilities: try to optimize the second solution to be viable also for small connections or , based on what I saw at runtime, pick the best implementation.

I went for the second option, which seemed to be the funnier one.

# The solution
I created a builder to provision and instance of either implementations. The builder would keep an [exponential weighted moving average](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average) of the size of the total per-connection-traffic. This is roughly the same algorithm TCP uses for RTT and Inter-Arrival Time variation estimation.

Funnily enough, it takes less characters to implement it than the characters I used to describe it:
```go
ewma := k * n + (1 - k) * ewma
```
Here `n` are the total bytes read from a connection on `Close`. `k` is just a constant in the code that makes the heuristic react slower or faster to size changes and I went for ½.

The hard part was to choose the threshold to switch implementation: I ran some benchmarks and found the tipping point where the streaming would start performing better better than the buffering, but I soon found out that the value heavily depended on the computer running the code.

# The overkill
In most cases running the benchmark on the author's laptop is enough to decide the constant to use, like [Go does  for `math/big`](https://github.com/golang/go/issues/25580) heuristics to choose the right algorithm to perform computation.

Needless to say, I don't like that approach, so I started putting together the tools go offered me. I didn't want to use makefiles or anything external, I didn't want my users to install or run anything exotic on their machines before being able to use my code, so I reasoned about **what was already available to the users compiling my library**.

Go is cross platform, so I could take nothing for granted, but on the other hand I needed something to get insights on the computational power I had available.

### Measuring
Benchmarking was relatively easy: go has [built-in benchmark standard libraries](https://golang.org/pkg/testing/#hdr-Benchmarks). 

After a couple of minutes dumbly staring at the doc I realized that I could run benchmarks from a non-testing build by calling [testing.Benchmark](https://golang.org/pkg/testing/#Benchmark), that returns a nice [testing.BenchmarkResult](https://golang.org/pkg/testing/#BenchmarkResult).

So I setup some `func(b *testing.B)` to closure an input value (e.g. the stub connection size), run a benchmark on both analysers and see which one performed better.

```go
type ConfigurableBenchmarker struct {
	Name string
	GetBench func(input []byte) func(b *testing.B)
}
```

An example of `ConfigurableBenchmarker` and how to use it:
```go
ConfigurableBenchmarker{
	Name: "Buffered",
	GetBench: func(input []byte) func(b *testing.B) {
		return func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				c := NewBufferedScanner(stubConnection{
					bytes.NewReader(input),
				})
				io.Copy(ioutil.Discard, c)
				c.Close()
			}
		}
	},
}

// doBench runs two ConfigurableBenchmarkers and returns whether the
// first one took less to execute.
func doBench(size int, aa, bb ConfigurableBenchmarker) bool {
	aaRes = testing.Benchmark(aa.GetBench(genInput(size)))
	bbRes = testing.Benchmark(bb.GetBench(genInput(size)))
	return aaRes.NsPerOp() < bbRes.NsPerOp()
}
```


With this building block I was empowered to do a binary search to see the size of the input at which one solution would start becoming better than the other one:
```go
func FindTipping(aa, bb ConfigurableBenchmarker, lower, upper int) (int,error) {
	lowerCond := doBench(lower, aa, bb)
	upperCond := doBench(upper, aa, bb)
	if lowerCond == upperCond {
		return 0, ErrTippingNotInRange
	}

	// Good old binsearch.
	tip = (lower + upper) / 2
	for tip > lower && tip < upper {
		tipCond := doBench(tip, aa, bb)
		if tipCond == lowerCond {
			lower = tip
		} else {
			upper = tip
		}
		tip = (lower + upper) / 2
	}
	return tip, nil
}
```


With this I could automatically detect the tipping point for my code on the current machine. Now I needed to run it. I could have put instructions in the README, but where's the fun in that?

### Go generate
The [`go generate`](https://blog.golang.org/generate) command allows you to parse comments with a particular syntax and run what's inside them.

The following comment makes go print a salutation when `go generate` is run.
```sh
//go:generate echo "Hello, World!"
```

So when a user `go get`s a package they can then `go generate` some code and `go build` it or link their source against it.

I wrapped the benchmarking code in a `generator.go` file, which runs benchmarks and writes the constant in a source file. It just formats a string with the number obtained from the benchmarks and writes it to a local file:
```go
const src = `// Code generated; DO NOT EDIT.

package main

const streamingThreshold = %d
`

func main() {
	tip := FindTipping(/* params */)
	// Omitted: open file "constants_generated.go" for writing in `f`
	fmt.Fprintf(f, src, tip)
}
```
Then I just needed to add a comment to the rest of the sources:
```sh
//go:generate go run generator.go
```
The machines I'm targeting must have `go` installed in order to compile my code. This means that I am not asking the user to add any external tool or dependency.

This was nice, but had a major issue: you can't have a package `main` and a package `analyse` in the same folder without using external build tools.

This is true unless you (ab)use [build tags](https://golang.org/pkg/go/build/#hdr-Build_Constraints): you can prevent a file from being considered for the build **before the `package` statement is read**.

So I changed my generator code to start with

```
// +build generate

package main
```
And the original code's comment to be
```
//go:generate go run generate.go -tags generate
```

Current structure:
```
analyse
├── analyse.go              ← Package analyse, with //go:generate directive
├── analyse_test.go         ← Package analyse, tests
├── constants_generated.go  ← Package analyse, generated
└── generate.go             ← Package main, behind a tag
```

So I can now `go get` or `git clone` my package, `go generate` it, and have it run optimized for my machine.
