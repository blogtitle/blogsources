---
title: "Sneaky race conditions and granular locks"
date: 2019-12-11T09:33:58+01:00
categories: ["Go", "Concurrency"]
tags: ["golang", "concurrency", "patterns", "engineering"]
authors: ["Rob"]
draft: true
---

Race conditions are the main problem you have to care about when writing concurrent code. Go provides high level primitives that make it easier to get concurrency and parallelism right, but those are often times not enough.

This is one of the reasons why Go has a [race detector](https://blog.golang.org/race-detector). You should always run your tests and, where possible, a portion of your production fleet with `-race` enabled. Beware that this might still not be enough: the detector only triggers when a race **manifests** and not all races are detected by it.

Today I'm not just going to talk about Go, I'm also going to compare its approach with the one that Rust uses and discuss the race conditions that both approaches fail to prevent. Rust approach to races is: if code contains unsynchronized access to shared data it should not compile.

As part of my job I dedicate a time to code reviews, and I kept tabs on the race conditions I spotted. After a while I noticed that the ones I found more or less followed similar evolution steps.

# First step: the naked mutex
The first step is to have a mutex that protects some kind of data, for this example I'll use integers and will call it `SharedInt` for simplicity.

```go
type SharedInt struct {
  mu sync.Mutex
  val int
}

func Answer(si *SharedInt) {
  si.mu.Lock()
  defer si.mu.Unlock()
  si.val == 42
}
```

# Second step: the encapsulation
After a while the code clutters with calls to `si.mu.Lock()` so the temptation is to create an helper method to set the value.

```go
type SharedInt struct {
  mu sync.Mutex
  val int
}

func (si *SharedInt) SetVal(val int) {
  si.mu.Lock()
  defer si.mu.Unlock()
  si.val = val
}
```

Of course that value needs to be read, so a getter is also defined:
```go
func (si *SharedInt) Val() int {
  si.mu.Lock()
  defer si.mu.Unlock()
  return si.val
}
```

This code works fine: there are writers that write values and readers that read them, no race condition as access to shared memory is synchronized.

After this step usually some time passes and this API is depended on by many packages and used by many programmers.

# Third step: the misuse
When some programmers new to the project look at the public API of the `SharedInt` type they see this:
```
type SharedInt struct
  SetVal(int)
  Val() int
```
And so they write the following code:
```go
func Increment(si *SharedInt) {
  si.SetVal(si.Val() + 1)
}
```
The race condition manifests between the call to `Val` and the one to `SetVal`.
At that point in time `si.val` is "stale": some other thread might have updated its value and we wouldn't know.  When we call the setter we pass a value based on the stale `v` we have. This will overwrite any update that happened between the call to `Val` and this line.

The access to memory is still synchronized so this code is "correct" for the race detector and for Rust type system, but it is not logically correct.

What's even worse is that during code review it is hard to spot, especially for people that come from Object Oriented Programming. The pattern "get -> update -> set" is extremely common and endorsed in the OOP paradigm.

