---
title: "I Am Switching to JS"
date: 2020-04-01T00:00:00+00:00
categories: ["Funny"]
tags: ["funny"]
authors: ["Rob"]
---

After 5 years of using Go I am finally moving on. Go has served me well and has been the best language I could have possibly used for the longest time, but it is now the moment for me to let it Go.

Over time Go has not failed to show me its limitations and its issues, to the point where I decided to switch to something more future proof and with a more thriving community.

I don't want to write this post as a list of things that pushed me away from my previous language, I find that kind of post sterile and of very little use for the readers, this is just a post on what I find great in JavaScript and what made me decide to switch.

# The type system and numbers
For the longest time JavaScript has had no integers. It only had some weird double that would sometimes well-behave as integer. Well, now that is no more, we have integers and arrays of integers that we can convert to each other's type, which makes it the language with the best numerical types.

```js
var sb = new ArrayBuffer(4)
var intArr = new Int32Array(sb)
intArr[0] = 1 << 16
// 65536
intArr[0]
// 65536
```

This is something I have been waiting for: exact arithmetics. It is incredible to think about a language having such a feature.

Beware, it doesn't stop here as this would just make JavaScript on par with many other languages. We can use a `Int32Array` as a `Int16Array` and it will work, no exception thrown and it will behave as expected:

```js
var sb = new ArrayBuffer(4)
var intArr = new Int32Array(sb)
intArr[0] = 1 << 16
// 65536
intArr[0]
// 65536
// Here I use the same backing buffer
var shortArray = new Int16Array(sb)
shortArray[0]
// 0
shortArray[1]
// 1
```
"Does this result depend on the endianness of the underlying architecture? If so, why are you on a big-endian architecture?" you might ask, and I would reply "Thanks for asking, that is a good question".

Moving on to another cool feature of these integers: it handles overflow as anyone would expect, unlike any other languages:
```JavaScript
var shortArr = new Int16Array(1)
var c = shortArr[0] = 1 << 15 // bonus: cool multiple assignment
c == shortArr[0]
// false
shortArr[0]
// -32768
c
// 32768
```

At this point I think I have shown the power of numbers in JavaScript, but I will add more just to make sure my point is clear.

You can use the modulus operator on floating point numbers and it does not misbehave:
```JavaScript
3.14 % 5
// 3.14
13.14 % 5
// 3.1400000000000006
```

It is also easy and immediate to sort an array of integers unlike many statically strongly typed languages, you don't even need to see the generic code underneath:
```JavaScript
[-2, -7, 0.0000001, 0.0000000006, 6, 10].sort()
// [-2, -7, 10, 1e-7, 6, 6e-10]
```

And syntax is always clear and intuitive:
```JavaScript
(1,2,3,4,5,6) === (2,4,6)
// true
```

Let's also say that you have a string that represents a number inputted by the user and you want to increment it by one. In many languages to do so you need to waste a lot of lines with annoying casts, while in JavaScript you can do it in a very elegant and readable way:
```JavaScript
var a = "41"
a += 1
// 411, wrong, unbalanced, weird.
var b = "41"
b -=- 1
// 42, with a beautifully symmetric operator, just perfect
```

The only surprising thing that I found and that is hard to keep in mind (and I have to get used to) is that when you deal with dates the months start from 0 while everything else starts from 1. There is no other surprising behavior to keep in mind that I know of.

# More on types
I find that the most beautiful part about JavaScript is indeed the type system so I would like to spend some more time on this with stuff that I found thanks to the JavaScript community on Twitter or while using this incredible language:

[Null coalescing](https://twitter.com/hashseed/status/1174238124756656128):
```JavaScript
~~!![[]]||__``&&$$++<<((""??''))**00==ಠಠ--//\\
// 1
```

[Want a banana? Beware of where you put your spaces](https://twitter.com/shadowcheets/status/1161012147884707840):
```JavaScript
('b'+'a'++'a'+'a').toLowerCase()
// Uncaught SyntaxError: Invalid left-hand side expression in postfix operation
('b' + 'a' + + 'a' + 'a').toLowerCase()
// "banana"
```

[Like to have many operators? JavaScript has got you covered](https://twitter.com/_jayphelps/status/1230538121743360001)
```JavaScript
// Concise code is best code
input ?? obj?.key ? 'yes' : 'no'
```

This one I particularly love: regexp ([see my post about them here](https://blogtitle.github.io/regexp-fun/)) and JavaScript joined together to create [something that will never get you bored](https://twitter.com/brianloveswords/status/1225459820746149889):
```JavaScript
var re = /a/g
re.test('ab')
// true
re.test('ab')
// false
re.test('ab')
// true
```

Operators are sometimes not commutative, so keep that in mind while you write your JavaScript:
```JavaScript
{property: "value"} && {property: "value"}
// {property: "value"}
Date() && Date()
// "Wed Apr 1 2020 00:01:00 GMT+0100 (Central European Standard Time)"
Date() && {property: "value"}
// {property: "value"}
{property: "value"} && Date()
// Uncaught SyntaxError: Unexpected token '&&'
```

Types might change under your fingers to help you stay awake during long coding nights:
```JavaScript
typeof([1][0])
// number
for(let i in [1]){console.log(typeof(i))}
// string
```

Even values might change by just observing them, just to make sure you don't sleep on the job:
```JavaScript
const x={
  i: 1,
  toString: function(){
    return this.i++;
  }
}

if(x==1 && x==2 && x==3){
  document.write("This will be printed!")
}
```

I could go on for another while on what is really nice about JavaScript being easy to read and easy to predict but I will stop with this last example that also takes the DOM into account:
```JavaScript
document.all
// HTMLAllCollection(359) [html.inited, head, …]
document.all == true
// false
document.all == false
// false
```

# A little deeper into the rabbit hole
This is a more serious section of this post. I usually like to dig deep in tools and languages I use, and what better way to learn things than ask colleagues and friends?

I recently started working with [Jan](https://twitter.com/terjanq) and he shared this amazing piece of JavaScript with me:
```JavaScript
(function({substr}){return substr.call})("")
// function call()
var x = (function({substr}){return substr.call})("")
// undefined
x.name
// "call"
x + ""
// "function call() {
//     [native code]
// }"
typeof x
// "function"
x("this is a string")
// TypeError: Function.prototype.call called on incompatible undefined
```

He didn't stop at that but also provided another amazing piece of code:
```JavaScript
(function(){return this}.call(1)) == (function(){return this}.call(1))
// false
```

# One last step: concurrency
Once I got to this point I was basically convinced to switch to JavaScript for good.

The last step would have been to check on my favorite stuff: concurrency and parallelism.

The complete lack of a standard library, an unpredictable type system, oddly-behaved operators, state-changing comparisons and non-telling compiler errors are the solid foundations I usually want to build my software on, but I needed to be sure also concurrency and parallelism would behave in a similar way.

Since I come from Go [it is immediate for me](https://play.golang.org/p/f_kSJ65nyba) to understand why this code prints three times '3':
```JavaScript
for (i = 1; i <= 2; ++i) {
  setTimeout(function(){
    console.log(i);
  }, 0);
}
```
But I have to admit the output of this code is a little less intuitive at first:
```JavaScript
console.log(0)
setTimeout(_ => console.log(1), 0)
requestAnimationFrame(_ => console.log(2))
Promise.resolve().then(_ => console.log(3))
console.log(4)
// 0
// 4
// 3
// 2
// 1
```
But then everything [becomes clearer](https://www.youtube.com/watch?v=cCOL7MC4Pl0) if you consider that we are calling two synchronous functions, we are scheduling a "Macro Task" we are spawning a "Micro Task" and we are requesting an animation frame.

With that I think concurrency in JavaScript is intuitive. What about parallelism?

Here is some code that I am going to be using, kindly provided by [`codediodeio`](https://github.com/codediodeio/async-await-pro-tips/blob/master/2-create-promise.ts)
```JavaScript
// The following code is to log how much time has passed:
const start = Date.now();
function log(v){console.log(`${v} \n Elapsed: ${Date.now() - start}ms`);}
// Here i define a CPU-intensive task that will block the current thread
function doWork(){
  for(let i = 0; i < 1000000000; i++); // note the semicolon
  return 'work done';
}
log('before work');
doWork();
log('after work');
// before work
//  Elapsed: 0ms
// after work
//  Elapsed: 624ms
```
This code was just to show how much time it takes to run the task in a synchronous way.

Let's now try to use `Promise`:
```JavaScript
function doWork(){
  return new Promise((resolve, reject) => {
    for(let i = 0; i < 1000000000; i++);
    resolve('work done');
  })
}
log('before work');
doWork().then(log);
log('after work');
// before work
//  Elapsed: 0ms
// after work
//  Elapsed: 637ms
// work done
//  Elapsed: 637ms
```
We have the same issue. Some might argue that the issue is that this is still running as a "Macro Task" or that we are blocking on it, so let's refactor it to run the entire loop in a "Micro Task":
```JavaScript
function doWork(){
  // Adding the `resolve` makes this run in a different kind of task
  return Promise.resolve().then(v =>  {
    for(let i = 0; i < 1000000000; i++);
    return 'work done';
  })
}
log('before work');
doWork().then(log);
requestAnimationFrame(()=>{log('time to next frame')});
log('after work');
// before work
//  Elapsed: 0ms
// after work
//  Elapsed: 1ms
// work done
//  Elapsed: 631ms
// time to next frame
//  Elapsed: 630ms
```
This seems to have addressed the issue if you just look at the first log lines, but it has not. It just moved the work to the next available execution slot on the main thread. We might have managed to run some of our synchronous code faster but the asynchronous one will still take its toll on our frame count ad the application will not be responding to the user for half a second.

This is something that has personally hit me at work: if you have intensive work to do in JavaScript, it will block the main thread and everything will become slow. You will find plenty of videos and tutorials that will tell you how to use `async` or `Promise` to address this issue and they will all be wrong. Those primitives only introduce **concurrency, not parallelism**. The only promises that will be able to run in parallel will be the one that are native in the browser like `fetch`, but **your code will never run in parallel**.

So how do you run code in parallel? You spawn a [web worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API), which is much like a thread in other languages. It is a cool new feature that introduces the much needed <s>race conditions</s> parallelism in JavaScript.

Using it is quite simple: we have to put the code that we want to run in parallel in a separate file, then we just instantiate the new worker and we can communicate with it via `postMessage`. This will give us real parallelism, which is nice and [well documented on MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers). Is it cumbersome? Sure, but it is there and people can create [libraries](https://github.com/GoogleChromeLabs/comlink) to use this primitive in a more user-friendly way. After all, [what is wrong with one more external dependency](https://en.wikipedia.org/wiki/Supply_chain_attack)?

Since JavaScript now has this amazing feature, it must provide some orchestration or synchronization primitive, right? Right. It does provide [`postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) and [`MessageChannel`](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel) which work in similar ways of `chan`, which is great. If you use just those you won't have race conditions and it is going to be very easy to reason about your concurrent code. Lovely.

**But** if you want something more responsive that doesn't have to trigger events here and there and wait for callbacks to be scheduled, if you want to **really** go faster, there are [`SharedArrayBuffers`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer). These are chunks of memory that you can share across threads and do [`Atomic`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics) operations on. No `Mutex`, no `select`, no `WaitGroup`. If you want to have those primitives you have to write your own [as I did](https://blogtitle.github.io/using-javascript-sharedarraybuffers-and-atomics/). HOW FUN IS THAT? This API even [returns values that are impossible to use correctly](https://github.com/tc39/ecma262/issues/1492)!

Who likes to have an easy job after all‽ (this is an interrobang symbol, which is not a JavaScript operator yet)

# Conclusions
JavaScript is now mature and provides everything I need: it has concurrency, parallelism and strongly typed variables. Moreover it spices up programming and makes everything fun by adding unpredictable and unfathomable behaviors right out of the box, so you don't get bored while debugging.

What's there not to like?

# Want more?
What to read more about the greatness of JavaScript? See [my previous post about it](https://blogtitle.github.io/lets-talk-about-javascript/)
