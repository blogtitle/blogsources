---
title: "Using JavaScript SharedArrayBuffers and Atomics"
date: 2019-03-22T12:42:05+01:00
categories: ["Concurrency"]
tags: ["javascript", "concurrency", "patterns", "engineering"]
authors: ["Rob"]
---

I'm not a [big fan of JavaScript](https://blogtitle.github.io/lets-talk-about-javascript/) but [I like parallel programming](https://blogtitle.github.io/categories/concurrency/) and JS has some interesting concepts of it.

In addition to [the event loop](https://www.youtube.com/watch?v=cCOL7MC4Pl0) I've recently got to know the concept of [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers). They are real threads that execute in parallel with the main thread, and are allowed to wait and block as much as they want.

Modern devices are going in the direction of having [a lot of FPS](https://support.techsmith.com/hc/en-us/articles/212255437-Apple-Device-Frame-Per-Second-Recording-Capability) and web developers want to maintain the user experience smooth. A good way to address it is to handle animations and UI-related tasks on the main thread and everything else in a worker.

This creates several issues of synchronization and communication: workers are only allowed to post messages to communicate, making coordination very burdensome and slow.

# A potential solution
Currently Chrome 67+ implements [`SharedArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer) and [`Atomics`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics).

The concept of a Shared Array Buffer is that you post a message to a worker, but instead of copying the content of the array you send **a reference to it**, so that multiple workers have visibility on a shared chunk of memory.

The concept of Atomics is that they provide operations that can happen "at once".

Here is an example of why atomics are important: when you perform an addition you issue several operations.
```js
a = a + 3;
// ↑ is equivalent to the following:
let reg = a;
reg += 3;
a = reg;
```
This is not an issue in normal JS because you are granted that **only one function executes at every given time**.

Introducing Workers and shared memory breaks this assumption, so someone might change the value of `a` while you are incrementing `reg`, and you end up overwriting their value. To address this you can use `Atomic.add`.

# A step forward, NaN backwards
While atomics give a great benefit, they are never easy to use. Moreover atomics are very low-level APIs and to express high-level behaviors it's usually needed to have multiple atomic operations interact with each other.

One common pattern in parallel programming is to have the concept of [critical sections](https://en.wikipedia.org/wiki/Critical_section) and [barriers](https://en.wikipedia.org/wiki/Barrier_(computer_science)). If you write Go you are probably familiar with `sync.Mutex` and `sync.WaitGroup` which implement those concepts.
The problem in JS is that Atomics are very low level and give the basic APIs, not the high level ones developers need.

Since I spent some time on those subjects I thought I might share some code and thoughts on how to leverage atomics to implement mutexes and wait groups.

# Mutexes
Mutexes are primitives that grant that only one thread can enter a critical section at the same time. A critical section is the portion of the code between a `lock` and an `unlock` call.

In details:

* Mutexes have 2 states: unlocked and locked.
* Calling `lock()` on a mutex should transition it in the locked state.
* Calling `lock()` on a mutex should block and wait if the mutex is locked by someone else.
* Calling `unlock()` on a mutex should transition it to the unlocked state.
* Calling `unlock()` on an unlocked mutex is not a valid operation.

For this we need the following atomic primitives:

* [`Atomics.compareExchange`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/compareExchange)
* [`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait)
* [`Atomics.notify`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/notify)

Let's take a look at the code:
```js
const locked = 1;
const unlocked = 0;

class Mutex {
  /**
   * Instantiate Mutex.
   * If opt_sab is provided, the mutex will use it as a backing array.
   * @param {SharedArrayBuffer} opt_sab Optional SharedArrayBuffer.
   */
  constructor(opt_sab) {
    this._sab = opt_sab || new SharedArrayBuffer(4);
    this._mu = new Int32Array(this._sab);
  }

  /**
   * Instantiate a Mutex connected to the given one.
   * @param {Mutex} mu the other Mutex.
   */
  static connect(mu) {
    return new Mutex(mu._sab);
  }

  lock() {
    for(;;) {
      if (Atomics.compareExchange(this._mu, 0, unlocked, locked) == unlocked) {
        return;
      }
      Atomics.wait(this._mu, 0, locked);
    }
  }

  unlock() {
    if (Atomics.compareExchange(this._mu, 0, locked, unlocked) != locked) {
      throw new Error("Mutex is in inconsistent state: unlock on unlocked Mutex.");
    }
    Atomics.notify(this._mu, 0, 1);
  }
}
```

This code works with just a backing shared `Int32` which is `this_mu[0]`. I'm not using an `Int8` because some atomic operations (like `wait`) only work on `Int32`.

When `lock` is called it tries to transition the mutex to the `locked` state and checks for success.
This is done via `compareExchange`, which means: "load the value and swap it with `locked` if and only if it is `unlocked`". `compareExchange` returns the old value of the variable which we can use to check if we managed to acquire the mutex or not. If we didn't, we go to sleep until someone calls `unlock`.

The sleep is implemented with a `wait`: we pass it a location to wait on and **the expected value**(locked). This bit is very important because if someone unlocked the mutex between our `compareExchange` and our `wait` we would end up sleeping forever.
Instead, if `wait` detects that the value is different from the one you expect, it will not suspend execution.

The `unlock` is fairly simple: we `compareExchange` a `locked` state with an `unlocked` state. If the mutex is in inconsistent state we throw, otherwise we notify one waiting thread that they can resume execution from a `lock` call. Note that if no one is waiting, the notify will be ignored.

# WaitGroup
WaitGroup is normally used to wait until all workers are done with their job and then read the cumulative output.

* Calling `add(n)` on a WaitGroup should change the counter by adding the given value
* Calling `done` on a WaitGroup is equivalent to calling `add(-1)`
* Calling `wait` on a WaitGroup should suspend execution until the counter is equal to `0`

Beware that `add` should always be called **before the work is started**, and this is why I added an initial value in the constructor. Calling `add` in the worker that calls `done` causes race conditions and might result in undesired early returns from `wait`.

To implement it I used the following primitives:

* [`Atomics.add`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/add)
* [`Atomics.load`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/load)
* [`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait)
* [`Atomics.notify`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/notify)

And here is the code:

```js
class WaitGroup {
  constructor(initial, opt_sab) {
    this._sab = opt_sab || new SharedArrayBuffer(4);
    this._wg = new Int32Array(this._sab);
    this.add(initial);
  }

  static connect(wg) {
    return new WaitGroup(0, wg._sab);
  }

  add(n) {
    let current = n + Atomics.add(this._wg, 0, n);
    if (current < 0) {
      throw new Error('WaitGroup is in inconsistent state: negative count.');
    }
    if (current > 0){
      return;
    }
    Atomics.notify(this._wg, 0);
  }

  done() {
    this.add(-1);
  }

  wait() {
    for (;;) {
      let count = Atomics.load(this._wg, 0);
      if (count == 0){
        return;
      }
      if (Atomics.wait(this._wg, 0, count) == 'ok') {
        return;
      }
    }
  }
}
```
As first thing the `add` implementation increments the counter with the given value. Since `add` returns the old value, we increment it with `n` and then reason on it as if it was frozen in time. We load the counter only once, and that grants coherence of this call.

The `notify` is called with the default amount of waiters, which is infinity.

The reason we are not checking if the counter is `0` in a wakeup from a wait is that a notify is called only in the event the counter reaches 0 during an add.

# Put it together
Here is a simple example of how to use those primitives:

`index.html`
```html
<script type="application/javascript" src="sync.js"></script>
<script type="application/javascript">
  const workers = [];
  const sab = new SharedArrayBuffer(4);
  const mu = new Mutex();
  const wg = new WaitGroup(size);
  // Warning: using a number that is too high might crash your browser.
  const size = 30;

  // Spawn a group of workers.
  for (let i = 0; i < size; i++) {
    workers.push(new Worker('worker.js'));
  }

  // Start the waiter
  const waiter = new Worker('waiter.js');
  waiter.postMessage({swg:wg, sc:sab});

  // Start the work.
  for (let w of workers){
    w.postMessage({swg:wg, smu:mu, sc:sab});
  }
</script>
```
`waiter.js`
```js
importScripts('sync.js');

self.addEventListener('message', function(e) {
  console.log('waiter started');
  // Deserialize data.
  const {swg, sc} = e.data;
  const wg = WaitGroup.connect(swg);
  const count = new Int32Array(sc);

  // Wait for workers to terminate.
  wg.wait();

  // The following lines will always execute last.
  console.log("final value:", count)
  console.log('waiter done');
}, false);
```

`worker.js`
```js
importScripts('sync.js');

self.addEventListener('message', function(e) {
  console.log('worker started');
  // Deserialize data.
  const {swg, smu, sc} = e.data;
  const wg = WaitGroup.connect(swg);
  const mu = Mutex.connect(smu);
  const count = new Int32Array(sc);

  // Do some stuff.
  const rnd = (Math.random() * Math.floor(10))|0;

  mu.lock();
  console.log("start of critical section");
  // This section does not require atomic operations, the mutex takes care of it.
  // This allows to do complex operations with the guarantee that no other worker
  // is in it. For example we could modify multiple sections of the array without
  // worrying that some might have changed before we are done.

  // This is a very simple example.
  count[0] = count[0] + rnd;
  // Load count[0] again, which won't have changed.
  console.log(`worker: ${rnd} ${count[0]}`)
  console.log("end of critical section");
  mu.unlock();

  // Simulate intensive computation.
  for (let i=0;i<100000;i++);

  // Signal termination.
  console.log('worker done');
  wg.done();
}, false);
```
