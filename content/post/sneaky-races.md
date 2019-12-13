---
title: "Sneaky race conditions and granular locks"
date: 2019-12-11T09:33:58+01:00
categories: ["Go", "Concurrency"]
tags: ["golang", "rust", "concurrency", "patterns", "engineering"]
authors: ["Rob"]
---

Race conditions are the main problem you have to care about when writing concurrent code. Go provides high level primitives that make it easier to get concurrency and parallelism right, but those are often times not enough.

This is one of the reasons why Go has a [race detector](https://blog.golang.org/race-detector). You should always run your tests and, where possible, a portion of your production fleet with `-race` enabled. Beware that this might still not be enough: the detector only triggers when a race **manifests** and not all races are detected by it.

Today I'm not just going to talk about Go, I'm also going to compare its approach with the one that Rust uses and discuss the race conditions that both approaches fail to prevent.

Rust approach to races is: if code contains unsynchronized access to shared data it should not compile. Beware that Rust definition of data race does not include all possible data races [as they explicitly document](https://doc.rust-lang.org/nomicon/races.html).

Important note: the Rust snippets are not written by me but by a [rustacean friend of mine](https://twitter.com/karroffel)

As part of my job I dedicate time to code reviews, and I kept tabs on the race conditions I spotted. After a while I noticed that the ones I found followed similar evolution steps and none of them would have been found or prevented by `go -race` or Rust type system.

The catastrophe happens in three simple steps.

# First step: the naked mutex
The first step is to have a mutex that protects some kind of data, for this example I'll use integers and will call it `SharedInt` for simplicity.

The following code defines a mutex-protected value and uses it in a function.

Every line in a critical section of a mutex or that performs a synchronized operation is marked with a lock emoji.

Go:
```go
type SharedInt struct {
  mu sync.Mutex
  val int
}

func Answer(si *SharedInt) {
  si.mu.Lock()
  defer si.mu.Unlock() // 🔒
  si.val = 42          // 🔒
}
```

Rust:
```rust
pub struct SharedInt {
  value: Mutex<isize>,
}
pub fn answer(si: &SharedInt) {
  let mut guard = si.value.lock().unwrap();
  *guard = 42; // 🔒
}
```
Rust does not need to unlock the mutex, when the guarded value goes out of scope the lock is automatically released.
Moreover the `unwrap` call makes `answer` panic if the mutex is "poisoned". Go mutexes do not support poisoning, but this is not relevant for this post. You can find out more [here](https://doc.rust-lang.org/std/sync/struct.Mutex.html).

# Second step: the encapsulation
After a while the code clutters with calls to `si.mu.Lock()` and `si.value.lock().unwrap();` so the temptation is to create an helper method to set the value:

```go
func (si *SharedInt) SetVal(val int) {
  si.mu.Lock()
  defer si.mu.Unlock() // 🔒
  si.val = val         // 🔒
}
```
Of course that value needs to be read at some point, so a getter is also defined:
```go
func (si *SharedInt) Val() int {
  si.mu.Lock()
  defer si.mu.Unlock() // 🔒
  return si.val        // 🔒
}
```
The Rust equivalent would be:
```rust
impl SharedInt {
  pub fn set_val(&self, val: isize) {
    let mut guard = self.value.lock().unwrap();
    *guard = val; // 🔒
  }

  pub fn val(&self) -> isize {
    let guard = self.value.lock().unwrap();
    *guard // 🔒
  }
}
```
This code works fine: there are writers that write values and readers that read them, we have no race condition.

After this step usually some time passes and this API is depended on by many packages and used by many programmers.

# Third step: the misuse
When some programmers new to the project look at the public API of the `SharedInt` type they see this:
```
SharedInt
  SetVal(int)
  Val() int
```
And so they write the following code:
```go
func Increment(si *SharedInt) {
  si.SetVal(si.Val() + 1)
}
```
```rust
pub fn increment(si: &SharedInt) {
    si.set_val(si.val() + 1);
}
```
Please take a little bit of time to look at the code above. Try to wear the shoes of the code reviewer for a minute and think if you would spot the problem in a ~400 lines long review.

# Explanation
The race condition manifests between the call to `Val` and the one to `SetVal`.

The code above can be seen as:
```go
func Increment(si *SharedInt) {
  v := si.Val() // 🔒
  v++           // 😱😱😱😱😱😱
  si.SetVal(v)  // 🔒
}
```
When we call the setter we pass a value based on the value we read. That value is stale at this point: we updated it without holding its lock, so the setter will overwrite any update that happened in the meantime.

The access to memory is still synchronized so this code is "correct" for the race detector and for Rust type system but it is not **logically** correct. I would say that you almost never want code like this to be seen as correct. Granular locks, like atomic operations, should be handled with extreme care or not at all.

What makes it even worse is **how hard it is to spot**, especially for people that, like me, come from Object Oriented Programming. The pattern "get -> update -> set" is extremely common and endorsed in the OOP paradigm. In Java [you always have getters and setters](https://www.w3schools.com/java/java_encapsulation.asp) for everything and in C# [you don't even know if you are accessing a field or you're calling a function](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/using-properties).

# Conclusion
Resist the temptation of hiding away your synchronization primitives as much as possible.

If you **have to encapsulate** make sure to provide at least one Read-Modify-Write primitive like `Add` or `CompareAndSwap`. You should do this **as soon as you encapsulate**, not when needed.
The sole presence of those methods will tip off your users that your type must be used as an atomic one.

Finally: **document how your code should be used and under which assumptions it works**. Do not make  your users read the code to see if you getter is doing something they should know about.

### Can we make better tools?

I think so, especially at runtime (the Go approach): when running code with `go test -race` the compiler could change the behavior of the code to **force rescheduling points on synchronization calls**. As far as I know Rust does not currently have this level of control on threads or a builtin race detector.

One example of this would be that every time a mutex is released during testing a call into the scheduler would be performed. This would make the races described in this post much more likely to happen and would not affect correct code.

As [Dmitry Vyukov points out](https://twitter.com/dvyukov/status/1205382645405954048?s=20), go already does [something similar](https://github.com/golang/go/blob/0497f911acbb33efbeac7271dc46e920fe26f4b8/src/runtime/proc.go#L4932-L4941) but this is not enough as it doesn't influence where scheduling points are emitted by the compiler, only the scheduler order.

Adding such a change would make incorrect code much more likely to not pass tests, which would at least be a good starting point.

# Want to know more?

[Here](https://blog.golang.org/race-detector) is an official post on the race detector.

[Here](https://doc.rust-lang.org/nomicon/races.html) is documentation on what is a race condition for Rust safety model.

[Here](http://www.1024cores.net/) is a blog on lock-free programming: the most extreme form of granular synchronization.
