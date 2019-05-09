---
title: "Let's talk more about JavaScript"
date: 2019-03-29T18:34:51+01:00
categories: ["Funny"]
tags: ["javascript", "funny", "rant"]
authors: ["Rob"]
draft: true
---

When I wrote [the previous post](TODO) I had recently picked up JavaScript after a long time not using it. The impact was awful.

Now it was almost 7 months since I last touched JavaScript and the overall satisfaction with my professional life was skyrocketing.

But then I did it again.

Here is my very personal second chapter of JavaScript whattefuckery.

# Types? No Types? What about both?

Js usually assigns any value to any variable, regardless the types.

```js
var a = 1;
a = "pluto is a planed";
```
This is valid code, and most of the time this is fine.

Now, you should know that JavaScript has Integers, not just doubles, and they have 52 bits precision.

Beware that when you force a cast to int, 20 bits are sacrificed to the Ancients, and you are stuck with 32 bits.
```js
var a = 1.4321 << 31;
// -2147483648
```
In this case the shift operator is only defined on integers, so this operation is equivalent to casting to a signed 32 bits integer type and shift left by 31.

This is still **almost** acceptable, so it gets worse.

You **can** declare your types:
```js
var b = new Int16Array(4);
b[0] = 4;
// prints 4
console.log(b[0]);
```
As any assignment in JavaScript, typed assignments return the assigned value:
```js
var b = new Int16Array(4);
// prints 4
console.log(b[0] = 4);
```
The return value can also be assigned:
```js
var b = new Int16Array(4);
var c = b[0] = 4;
// prints 4
console.log(c);
```
Now, one would expect that whatever is the value assigned to b[0] in the code above, `c === b[0]` will always be true.

But it's not.
```js
var b = new Int16Array(4);
var c = b[0] = 32768;
// prints 32768 -32768 false
console.log(c, b[0], c === b[0]);
```

```js
var b = new Int16Array(4);
var c = b[0] = "Oh ${deity} why??";
// prints Oh ${deity} why?? 0 false
console.log(c, b[0], c === b[0]);
```
#  Just sort your s\*\*t

Sorting is a pretty straightforward operation, isn't it?
```js
// prints [-2, -7, 0.00001, 10, 6, 6e-13]
[-2, 10, 6, 0.00001, 0.0000000000006, -7].sort()
```

# Closure scope

This is just another painful instance of hoisting.

This prints eleven times "11":
```js
for (var i = 0; i <= 10; i++) {
    setTimeout(()=>console.log(i), 0)
}
```
This, instead, prints numbers from 1 to 10, as expected:
```js
for (let i = 0; i <= 10; i++) {
  setTimeout(()=>console.log(i),0)
}
```
# Broken promises
Promises and `async`/`await` were added to make the callback hell more human friendly. And they serve that purpose.

One thing they don't do and that developers still somehow are convinced they do is to add parallelism to JavaScript.

The only things that execute in parallel in the browser when you use promises are native promises.So yeah, if you fetch two things in parallel, they will run in parallel indeed:
```js
var a = fetch('https://empijei.science');
var b = fetch('https://empijei.science');
// Assuming each fetch will take 500 ms to retrieve content,
// this will take a little bit more than 500ms total, and not 1 full second:
Promise.all([a, b]);
```
But the trick here is that `fetch` code is not user-provided, so it is allowed to run in parallel.

The same holds true for timeouts:
```js
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
Promise.all([delay(1000),delay(1000),delay(1000),delay(1000)])
  .then(_=>console.log("1s later"))
```
But again, the timeout code is not user-provided code.

Let's use this code:
```js
function delay(ms) {
  return new Promise(resolve => {
    // Simulate computation
    for (let i=0; i<ms*1000000; i++);
    resolve()
  });
}
// This takes 3 times more than the code below to run:
Promise.all([delay(1000),delay(1000),delay(1000)]).then(_=>console.log("foo"))
Promise.all([delay(1000)]).then(_=>console.log("foo"))
```
This has a problem: the code is actual running the loops on the main thread, when the promises are constructed.

So someone might tell you that you need to change your code like this:
```js
function delay(ms) {
  return Promise.resolve().then(_=> {
    // Simulate computation
    for (let i=0; i<ms*1000000; i++);
  });
}
// This takes 3 times more than the code below to run, no improvements here.
Promise.all([delay(1000),delay(1000),delay(1000)]).then(_=>console.log("foo"))
Promise.all([delay(1000)]).then(_=>console.log("bar"))
```
