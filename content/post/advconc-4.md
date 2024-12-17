---
title: "Go advanced concurrency patterns: part 4 (unlimited buffer channels)"
date: 2024-12-16T22:19:23+01:00
categories: ["Go", "Concurrency"]
tags: ["golang", "concurrency", "patterns", "engineering"]
authors: ["Rob"]
---

# Preface

It's been almost 6 years since I added to this series, but I thought it was time
to resume working on it.

Since "perfect is the opposite of done" I thought I would talk about a simple pattern
that I haven't covered yet.

A couple of years ago I worked on [a toy library](https://pkg.go.dev/github.com/empijei/channels)
to see if the new-ish generics proposal would finally allow to express some of the
most common channel patterns in Go.

Turns out it did, so I stopped working on it and decided to wait for the community
or the standard library to actually come up with something better.

I'm still not convinced by anything that came out since, so here I am: sharing some
of the learnings from my experiments so that you can write your own.

# Disclaimer

Probably one of the most common requests that I get from new gophers is "how do I
create a channel with unlimited buffer?". The answer to that is "you don't".

The main reason for this is that unlimited channels make code a lot harder to debug
and issues that would otherwise be symptomatic become silent until a whole system
comes down.

You can find more [in this conversation](https://github.com/golang/go/issues/20352).

That said, there are some rare cases in which such a feature would allow a system
to absorb bursty traffic without blocking, or where quickly deallocating or reusing
producers is much more efficient than just keeping a buffer of what they produced.

For those rare situations the following code can be used.

# Solutions

The nicest API I could think of is in the form of:

```go
func BufferChan[T any](in <-chan T) (out <-chan T)
```

This allows us to still deal with channels without introducing iterators or things
that don't work well with built in statements like `select` or that can't easily
be cancelled with `context`.

The semantics would be:

- guarantee that all values received on `in` will be emitted on `out`
- when `in` is closed and all potentially buffered values have been emitted `out`
  will be closed.

If you are unfamiliar with the `<-chan` type declaration, in this case it means
that this function guarantees to never _send_ on `in` and prevents its callers
to ever _send_ on `out` (closing a channel is a _send_ operation).

## When order doesn't matter and the operation is short-lived

If producers are much faster than consumers and so `send` operations outnumber
`receive` operations, order rarely matters.

For all of those cases where operations are short lived we can use this simple solution:

```go
func BufShortLived[T any](in <-chan T) <-chan T {
 var buf []T
 out := make(chan T)
 go func() {
  defer close(out)         // Close once we are done
  for v := range in {      // Process all input
   select {
   case out <- v:          // Try to send
   default:                // If we fail
    buf = append(buf, v)   // Add to buffer
   }
  }
  for _, v := range buf {  // Input has been closed, emit buffered values.
   out <- v
  }
 }()
 return out
}
```

This buffers all values that were produced and that the consumers weren't fast enough
to consume, and emits them back once the channel is closed.

Since the buffer is only ever released at the end of operations, this solution might not be for you.

Please still consider it as it is very simple to read and test.

# When order does matter, and the operation lasts a while

I still have to see a reasonable use case for ordered unlimited channels,
but since there are some people, especially from other languages communities,
that insist this is a fundamental feature to have, we might as well talk about it.

_I cannot stress enough how much harder this will make your life if you use this in production code_,
but this is a blogpost, not a cop, so you do you.

When reading this code consider that operations on `nil` channels block forever.
A `select` case with a `nil` channel is, therefore, never selected.

Here `nout` will be `nil` _if and only if_ there are no buffered values to send,
otherwise it will be equal to `out`. We use it to switch on and off its `case`.

```go
func BufLongLivedOrdered[T any](in <-chan T) <-chan T {
 out := make(chan T)
 go func() {
  defer close(out)
  var (
   queue []T     // This is our buffer
   next  T       // This is the next value to be sent
   nout  chan T  // This is a nil-lable reference to "out"
  )
  for in != nil || nout != nil { // Until both "in" and "out" have been disabled.
   select {
   case v, ok := <-in: // If we receive
    if !ok {           // and "in" has been closed
     in = nil          // disable this case.
     continue
    }
    if nout == nil { // If "out" is disabled
     nout = out      // enable it
     next = v        // and prepare the value to send.
    } else {
     queue = append(queue, v) // otherwise we have already stuff queued, append
    }
   case nout <- next:    // If we send a value
    if len(queue) == 0 { // and ran out of buffer
     nout = nil          // disable this case
     continue
    }
    next = queue[0] // otherwise prepare next value to send.
    queue = queue[1:] // Consume the buffer.
   }
  }
 }()
 return out
}
```

What I like about this solution is that it doesn't make use of mutexes or conditions
to protect the buffer or wake up waiters, but solely relies on channel semantics.

Since only one goroutine is ever going to touch `in` and `out` we don't need to guard
against potential race conditions.

## Improving on it

One minor issue with the solution above is that we give equal priority to sending
buffered values and receiving new values.

This might, in turn, cause buffers to grow more than necessary. To avoid this, a
preamble that is a copy of the second `case` can be added at the beginning of the for,
just with a `default` to not block on it:

```go
  for in != nil || nout != nil { // Same as before
   select { // Added block
   case nout <- next:
    if len(queue) == 0 {
     nout = nil
     continue // This cannot be a break statement, we need to check the exit condition
    }
    next = queue[0]
    queue = queue[1:]
    continue
   default:
      // If we get here no consumers were ready, or no value was buffered,
      // just continue.
   }
   // Same select as above
   select {
```

This will make the code above always try to send before it tries to both send
and receive at once. In some rare cases this might actually reduce the buffer size.

## Full code

If you just came here for the code, here it is:

```go
func BufLongLived[T any](in <-chan T) <-chan T {
	out := make(chan T)
	go func() {
		defer close(out)
		var (
			queue []T
			next  T
			nout  chan T
		)
		for in != nil || nout != nil {
			select {
			case nout <- next:
				if len(queue) == 0 {
					nout = nil
                    continue
				}
				next = queue[0]
				queue = queue[1:]
				continue
			default:
			}
			select {
			case v, ok := <-in:
				if !ok {
					in = nil
					continue
				}
				if nout == nil {
					nout = out
					next = v
				} else {
					queue = append(queue, v)
				}
			case nout <- next:
				if len(queue) == 0 {
					nout = nil
					continue
				}
				next = queue[0]
				queue = queue[1:]
			}
		}
	}()
	return out
}
```

# Conclusions

In 31 lines of code we implemented a channel operator that gives us a new ordered,
unlimited buffer channel from an existing one.

I found this quite tricky to get right and I still think that there are rare use
cases for this, in fact, in my career I have yet to find one. I think you're better
off trying to find a good size for your buffers or reconsider your architecture
before you resort to this.

That said, I hope it was useful, happy hacking!
