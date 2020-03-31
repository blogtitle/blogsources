---
title: "Go advanced concurrency patterns: part 3 (channels)"
date: 2019-12-05T20:07:04+01:00
categories: ["Go", "Concurrency"]
tags: ["golang", "concurrency", "patterns", "engineering"]
authors: ["Rob"]
---

Today I'm going to dive into the expressive power of Go channels and the `select` statement. To demonstrate how much can be implemented with just those two primitives I will rewrite the `sync` package from scratch.

In doing so I'm just going to accept some compromises:

* This post is about expressive power, not speed. The examples will correctly express the primitives but might not be as fast as the real ones.
* Channels, unlike slices, don't have a fixed size equivalent type. While you can have a `[4]int` type in Go you can't have a `chan int` of size 4, so I won't have valid zero values and all types will require a constructor call. There is [a proposal open](https://github.com/golang/go/issues/28366) to address this issue but this is beyond the scope of this post.
* Most types will be just channels but nothing prevents you from hiding those channels in an opaque struct to avoid misuse. Since this is a blogpost and not an actual library I'm going to use bare channels for brevity and clarity.
* I will not explain what the primitives are supposed to do but I'll link to their official doc.

That said, let's start!

# Once

[Once](https://golang.org/pkg/sync/#Once) is a fairly simple primitive: the first call to `Do(func())` will cause all other concurrent calls to block until the argument of `Do` returns. After this happens all blocked calls and successive ones will do nothing and return immediately.

This is useful for lazy initialization and singleton instances.

Let's take a look at the code:

```go
type Once chan struct{}

func NewOnce() Once {
	o := make(Once, 1)
	// Allow for exactly one read.
	o <- struct{}{}
	return o
}

func (o Once) Do(f func()) {
	// Read from a closed chan always succeedes.
	// This only blocks during initialization.
	_, ok := <-o
	if !ok {
		// Channel is closed, early return.
		// f must have already returned.
		return
	}
	// Call f.
	// Only one goroutine will get here as there is only one value
	// in the channel.
	f()
	// This unleashes all waiting goroutines and future ones.
	close(o)
}
```

# Mutex

For [Mutex](https://golang.org/pkg/sync/#Mutex) I'm going to make two little digressions:

* A Semaphore of size N allows at most N goroutines to hold its lock at any given time. Mutexes are special cases of Semaphores of size 1.
* Mutexes might benefit from a `TryLock` method

The `sync` package does not provide semaphores or triable locks but since we are trying to prove the expressive power of channels and `select` let's implement both.

```go
type Semaphore chan struct{}

func NewSemaphore(size int) Semaphore {
	return make(Semaphore, size)
}

func (s Semaphore) Lock() {
	// Writes will only succeed if there is room in s.
	s <- struct{}{}
}

// TryLock is like Lock but it immediately returns whether it was able
// to lock or not without waiting.
func (s Semaphore) TryLock() bool {
	// Select with default case: if no cases are ready
	// just fall in the default block.
	select {
	case s <- struct{}{}:
		return true
	default:
		return false
	}
}

func (s Semaphore) Unlock() {
	// Make room for other users of the semaphore.
	<-s
}
```

If we now want to implement a Mutex based on this we can do the following:

```go
type Mutex Semaphore

func NewMutex() Mutex {
	return Mutex(NewSemaphore(1))
}
```

# Read-Write Mutexes

[RWMutex](https://golang.org/pkg/sync/#RWMutex) is a slightly more complicated primitive: it allows an arbitrary amount of concurrent read locks but only a single write lock at any given time. It is also guaranteed that if someone is holding a write lock no one should be able to have or acquire a read lock.

The standard library implementation also grants that if a write lock is attempted, further read locks will queue up and wait to avoid starving write lock. We are going to relax this condition for brevity. There is a way to to this by using a 1-sized channel as mutex to protect the RWLock state, but it is a boring example to show. This implementation will starve writers if there is always at least one reader.

Note: This code is inspired by an implementation of this concept by Bryan Mills.

RWMutex has three states: free, with a writer and with readers. This means we need two channels: when the mutex is free we'll have both channels empty; when there is a writer we'll have a channel with an empty struct in it; when there are readers we'll have a value in both channels, one of which will be the count of the readers.

```go
type RWMutex struct {
	write   chan struct{}
	readers chan int
}

func NewLock() RWMutex {
	return RWMutex{
		// This is used as a normal Mutex.
		write:   make(chan struct{}, 1),
		// This is used to protect the readers count.
		// By receiving the value it is guaranteed that no
		// other goroutine is changing it at the same time.
		readers: make(chan int, 1),
	}
}

func (l RWMutex) Lock() { l.write <- struct{}{} }
func (l RWMutex) Unlock() { <-l.write }

func (l RWMutex) RLock() {
	// Count current readers. Default to 0.
	var rs int
	// Select on the channels without default.
	// One and only one case will be selected and this
	// will block until one case becomes available.
	select {
	case l.write <- struct{}{}: // One sending case for write.
		// If the write lock is available we have no readers.
		// We grab the write lock to prevent concurrent
		// read-writes.
	case rs = <-l.readers: // One receiving case for read.
		// There already are readers, let's grab and update the
		// readers count.
	}
	// If we grabbed a write lock this is 0.
	rs++
	// Updated the readers count. If there are none this
	// just adds an item to the empty readers channel.
	l.readers <- rs
}

func (l RWMutex) RUnlock() {
	// Take the value of readers and decrement it.
	rs := <-l.readers
	rs--
	// If zero, make the write lock available again and return.
	if rs == 0 {
		<-l.write
		return
	}
	// If not zero just update the readers count.
	// 0 will never be written to the readers channel,
	// at most one of the two channels will have a value
	// at any given time.
	l.readers <- rs
}
```

`TryLock` and `TryRLock` methods could be implemented as I did in the previous example by adding default cases to channel operations:

```go
func (l RWMutex) TryLock() bool {
	select {
	case l.write <- struct{}{}:
		return true
	default:
		return false
	}
}

func (l RWMutex) TryRLock() bool {
	var rs int
	select {
	case l.write <- struct{}{}:
	case rs = <-l.readers:
	default:
		// Failed to lock, do not update anything.
		return false
	}
	rs++
	l.readers <- rs
	return true
}
```

# Pool
[Pool](https://golang.org/pkg/sync/#Pool) is used to relieve stress on the garbage collector and re-use objects that are frequently allocated and destroyed.

The standard library uses many techniques for this: thread local storage; removing oversized objects and generations to make objects survive in the pool only if they are used between two garbage collections.

We are not going to provide all of these functionalities: we don't have access to thread local storage nor garbage collection information, nor we care. Our pool will be fixed size as we just want to express the semantic of the type, not the implementation details.

One utility that we are going to provide that Pool does not have is a cleanup function.

It is very common when using Pool to not clear up the objects that come out of it, thus causing re-use of non-zeroed memory, which can lead to nasty bugs and vulnerabilities. Our implementation is going to guarantee that a cleaner function is called if and only if the returned object is recycled.

This could, for example, be used on slices to re-slice to 0 length or on structs to zero all the fields.

If a cleanup is not specified we will not call it.

```go
// Item is any type, I gave it a name for readability.
type Item = interface{}

type Pool struct {
	buf   chan Item
	alloc func() Item
	clean func(Item) Item
}

func NewPool(size int, alloc func() Item, clean func(Item) Item) *Pool {
	return &Pool{
		buf:   make(chan Item, size),
		alloc: alloc,
		clean: clean,
	}
}
func (p *Pool) Get() Item {
	select {
	case i := <-p.buf:
		if p.clean != nil {
			return p.clean(i)
		}
		return i
	default:
		// Pool is empty, allocate new instance.
		return p.alloc()
	}
}
func (p *Pool) Put(x Item) {
	select {
	case p.buf <- x:
	default:
		// Pool is full, garbage-collect item.
	}
}
```

One example usage of this would be:
```go
	p := NewPool(1024,
		func() interface{} {
			return make([]byte, 0, 10)
		},
		func(i interface{}) interface{} {
			return i.([]byte)[:0]
		})
```

# Map
[Map](https://golang.org/pkg/sync/#Map) is not very interesting for this post as it is semantically equivalent to a `map` with a `RWMutex` on it. The `Map` type is basically a dictionary optimized for use cases that foresee to have many more reads than writes. It is just a technique to make some specific use cases faster and I will thus skip it as today we don't care about speed.

# WaitGroup

[WaitGroup](https://golang.org/pkg/sync/#WaitGroup) somehow always ends up being one of the most complicated primitives to implement. This also happened [in my JavaScript's Atomic post](https://blogtitle.github.io/using-javascript-sharedarraybuffers-and-atomics/) where I put a race condition in the `WaitGroup` (have fun finding it).

Wait groups allow for a variety of uses but the most common one is to create a group, `Add` a count to it, spawn as many goroutines as that count and then wait for all of them to complete. Every time a goroutine is done running it will call `Done` on the group to signal it has finished working. Wait groups can reach the "0" count by calling `Done` or calling `Add` with a negative count (even greater than -1).
When a group reaches 0 all the waiters currently blocked on a `Wait` call will resume execution. Future calls to `Wait` will not be blocking.

One little known feature is that wait groups can be re-used: `Add` can still be called after the counter reached 0 and it will put the wait group back in the blocking state.

This means we have a sort of "generation" for each given wait group:

* a generation begins when the counter moves from 0 to a positive number
* a generation ends when the counter reaches 0
* when a generation ends all waiters of that generation are unblocked.

Let's see it in code:

```go
type generation struct {
	// A barrier for waiters to wait on.
	// This will never be used for sending, only receive and close.
	wait chan struct{}
	// The counter for remaining jobs to wait for.
	n int
}

func newGeneration() generation {
	return generation{ wait: make(chan struct{}) }
}
func (g generation) end() {
	// The end of a generation is signalled by closing its channel.
	close(g.wait)
}

// Here we use a channel to protect the current generation.
// This is basically a mutex for the state of the WaitGroup.
type WaitGroup chan generation

func NewWaitGroup() WaitGroup {
	wg := make(WaitGroup, 1)
	g := newGeneration()
	// On a new waitgroup waits should just return, so
	// it behaves exactly as after a terminated generation.
	g.end()
	wg <- g
	return wg
}

func (wg WaitGroup) Add(delta int) {
	// Acquire the current generation.
	g := <-wg
	if g.n == 0 {
		// We were at 0, create the next generation.
		g = newGeneration()
	}
	g.n += delta
	if g.n < 0 {
		// This is the same behavior of the stdlib.
		panic("negative WaitGroup count")
	}
	if g.n == 0 {
		// We reached zero, signal waiters to return from Wait.
		g.end()
	}
	// Release the current generation.
	wg <- g
}

func (wg WaitGroup) Done() { wg.Add(-1) }

func (wg WaitGroup) Wait() {
	// Acquire the current generation.
	g := <-wg
	// Save a reference to the current waiting chan.
	wait := g.wait
	// Release the current generation.
	wg <- g
	// Wait for the chan to be closed.
	<-wait
}
```

# Condition
[Cond](https://golang.org/pkg/sync/#Cond) is the most controversial type in the sync package. I think it's a dangerous primitive and it is too easy to use incorrectly. I never use it because I don't trust myself to use it right and during code review I always suggest to use other primitives instead.
Even Bryan Mills (who is in the Go team and works a lot on `sync` primitives) got to the point of [proposing to remove it](https://github.com/golang/go/issues/21165).

I suppose the most important reason for `Cond` to exist is that channels cannot be re-opened to broadcast twice but I am not sure the benefit is worth the cost.

Even not considering the fact that this is error prone it also doesn't work well with channels ([issue about this](https://github.com/golang/go/issues/16620)). It is, for example, currently impossible to select over a `Cond` and context cancellation: it requires some wrapping and additional goroutines, which are expensive and might be leaked.

There is even more weirdness in this API: it requires its users to do part of the synchronization themselves. Quoting the doc:

> Each Cond has an associated Locker L (often a *Mutex or *RWMutex), which must be held when changing the condition and when calling the Wait method.

This means that we don't have to care about the user changing the `L` field of Cond or racing on `Wait` calls, but there might be races on `Broadcast` and `Signal` which we'll address.

This gets even worse if we consider the documentation on `Wait`:

> Wait atomically unlocks c.L and suspends execution of the calling goroutine. After later resuming execution, Wait locks c.L before returning.
> Because c.L is not locked when Wait first resumes, the caller typically cannot assume that the condition is true when Wait returns. Instead, the caller should Wait in a loop.

To implement this odd primitive I am going to use a channel of channels.

```go
type Locker interface {
	Lock()
	Unlock()
}

type barrier chan struct{}

type Cond struct {
	L  Locker
	bs chan barrier
}

func NewCond(l Locker) Cond {
	c := Cond{
		L:  l,
		bs: make(chan barrier, 1),
	}
	// Waits will block until signalled to continue.
	c.bs <- make(barrier)
	return c
}

func (c Cond) Broadcast() {
	// Acquire barrier.
	b := <-c.bs
	// Release all waiters.
	close(b)
	// Create a new barrier for future calls.
	c.bs <- make(barrier)
}

func (c Cond) Signal() {
	// Acquire barrier.
	b := <-c.bs
	// Release one waiter if there are any waiting.
	select {
	case b <- struct{}{}:
	default:
	}
	// Release barrier.
	c.bs <- b
}

// According to the doc we have to perform two actions atomically:
// * Call Unlock
// * Suspend execution
// To do so we receive the current barrier, call Unlock while we still
// hold it and release it. This guarantees that nothing else has happened
// in the meantime.
// After this operation we wait on the barrier we received, which
// might not reflect the current one (as intended).
func (c Cond) Wait() {
	// Acquire barrier.
	b := <-c.bs
	// Unlock while in critical section.
	c.L.Unlock()
	// Release barrier.
	c.bs <- b
	// Wait for release on the value of barrier that was valid during
	// the call to Unlock.
	<-b
	// We were unblocked, acquire lock.
	c.L.Lock()
}
```

# Conclusion
Go channels and `select` combined are extremely expressive and allow to create high level synchronization and orchestration primitives in a very expressive way. I think many of the design choices, like having limited size channels or selects that can receive and send at the same time, give Go builtin types something that is rarely seen in other languages.

# Want to learn more?

* [Here](https://blogtitle.github.io/categories/concurrency/) you can find my other posts on advanced concurrency.

* [Here](https://drive.google.com/file/d/1nPdvhB0PutEJzdCq5ms6UI58dp50fcAN/view) you can find a very interesting presentation on rethinking concurrency patterns by Bryan Mills.

