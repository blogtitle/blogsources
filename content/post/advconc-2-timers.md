---
title: "Go Advanced concurrency patterns: part 2 (timers)"
date: 2019-02-25T22:56:23+01:00
categories: ["Go", "Concurrency"]
tags: ["golang", "concurrency", "patterns", "engineering"]
authors: ["Rob"]
draft: true
---
 As I stated [in my previous article](https://blogtitle.github.io/go-advanced-concurrency-patterns-part-1/) timers are hard to use the right way so here are some handful tips.

# Preface
If you don't think dealing with time and timers while also juggling goroutines is hard, here are some of the **many** bugs related to time.Timer:

* [time: Timer.Reset is not possible to use correctly #14038](https://github.com/golang/go/issues/14038)
* [time: Timer.C can still trigger even after Timer.Reset is called #11513](https://github.com/golang/go/issues/11513)
* [time: document proper usage of Timer.Stop #14383](https://github.com/golang/go/issues/14383)

If you are not satisfied, in the following code there are a deadlock and a race condition:
```go
	tm := time.NewTimer(1)
	tm.Reset(100 * time.Millisecond)
	<-tm.C
	if !tm.Stop() {
		<-tm.C
	}
```

This just has a deadlock:
```go
func toChanTimed(t *time.Timer, ch chan int) {
	t.Reset(1 * time.Second)
	defer func() {
		if !t.Stop() {
			<-t.C
		}
	}()
	select {
	case ch <- 42:
	case <-t.C:
	}
}
```

# time.Tick
This is just something you should never use unless you plan to carry the returned chan around and keep using it for the entire lifetime of the program.

As the documentation states:

> the underlying Ticker cannot be recovered by the garbage collector; it "leaks".

Please use this with caution.

# time.After
This is basically the same concept of `Tick` but instead of hiding a `Ticker`, hides a `Timer`. This is slightly better because once the timer will fire, it will be collected. Please note that timers use 1-buffered channels, so they can fire even if no one is receiving anymore.

As above, if you care about performance and you want to be able to cancel the timer, you should not use `After`.

# time.Timer
I find this to be quite an odd API for Go, and it is rather unusual: `NewTicker(Duration)` returns a `*Timer`, which has an **exported** chan field called `C` that is the interesting value for the caller.

The oddity is that usually in Go exported fields mean that the user can both get and **set** them, while here setting `C` doesn't mean anything. It's quite the opposite: setting `C` and resetting the Timer will still make the runtime deliver messages in the channel that was the previous `C`.

That said, `Timer` has a lot more interesting oddities. Here is an overview on the API:

* AfterFunc(d Duration, f func()) *Timer
* NewTimer(d Duration) *Timer
* (*Timer).Stop(bool)
	* (*Timer).Reset(d Duration) bool

Four very simple funcitons, two of which are constructors, what could possibly go wrong?

### AfterFunc
The official doc:

> AfterFunc waits for the duration to elapse and then calls f in its own goroutine. It returns a Timer that can be used to cancel the call using its Stop method.

While that is correct, be careful: when calling `Stop`, if `false` is returned, it means that stopping failed and the function was already started. This does not say anything about that function having returned, for that you need to add some manual coordination:

```go
done := make(chan struct{})
f := func() {
	doStuff()
	close(done)
}
t := time.AfterFunc(1*time.Second, f)
if !t.Stop() {
	<-done
}
```

### NewTimer
> NewTimer creates a new Timer that will send the current time on its channel after at least duration d.

There is no way to construct a valid Timer without starting it. This means that if you need to construct one for future re-use, you either do it lazily or you have to create and stop it, which translates to this code:
```go
t := time.NewTimer(0)
if !t.Stop() {
	<-t.C
}
```
You **have to** read from the channel: if the timer had time to fire between the `New` and the `Stop` calls, there will be a value in the channel, and future reads will be wrong.

### Stop
> Stop prevents the Timer from firing. It returns true if the call stops the timer, false if the timer has already expired **or** been stopped.

The bold **or** in the sentence above is very important. All examples for `Stop` in the doc show the very same snippet:
```go
if !t.Stop() {
	<-t.C
}
```
The problem is that the **or** means that this pattern is valid 0 or 1 times. It is not valid if someone else already drained the channel, and it is not valid to do this multiple times without calling `Reset` in between. This summarized means that `Stop` is safe to call that way if and only if no reads from the channel have been performed, including ones caused by the code above.

All this is stated in the doc by the sentence below:

> For example, assuming the program has not received from t.C **already**:

Moreover, the pattern above is not thread safe, as the value returned by `Stop` might already be stale when draining the channel is attempted, and this would cause a deadlock as two goroutines would try to drain `C`.

### Reset
This is even more interesting. The documentation is quite long, and you can find it [here](https://golang.org/pkg/time/#Timer.Reset).

One funny extract is:

> Note that it is not possible to use Reset's return value correctly, as there is a race condition between draining the channel and the new timer expiring. **Reset should always be invoked on stopped or expired channels**

The doc says the the right way to use `Reset` is the following one:
```go
if !t.Stop() {
	<-t.C
}
t.Reset(d)
```
You cannot use `Stop` nor `Reset` concurrently with other receives from the channel, and in order for the value sent on `C` to be valid, `C` should be drained exactly once before each `Reset`.

Resetting a timer without draining it will make the runtime drop the value, as `C` is of size 1 and the runtime performs [a lossy send on it](https://golang.org/src/time/sleep.go?s=#L134).

### Putting it together
* `Stop` is safe only after `New` and `Reset`.
* `Reset` is only valid after `Stop`.
* Received value is valid only if channel is drained after each `Stop`.
* The channel should be drained if and only if the channel has not been read yet.

So here is an example of a correct usage of a re-usable timer, which fixes one of the examples at the beginning of this post:
```go
func toChanTimed(t *time.Timer, ch chan int) {
	t.Reset(1 * time.Second)
	// No defer, as we don't know which
	// case will be selected

	select {
	case ch <- 42:
	case <-t.C:
		// C is drained, early return
		return
	}

	// We still need to check the return value
	// of Stop, because t could have fired
	// between the send on ch and this line.
	if !t.Stop() {
		<-t.C
	}
}
```
This ensures the timer is ready to be re-used after `toChanTimed` returns.

# time.Ticker
// Todo unread ticks will be dropped(except for the first)


// TODO
They both rely on runtimeTimer, but one passes a period of zero.
