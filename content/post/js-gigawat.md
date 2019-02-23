---
title: "Let's talk about JavaScript"
date: 2018-08-11T19:20:13+02:00
categories: ["Funny"]
tags: ["javascript", "funny", "rant"]
authors: ["Rob"]
---

# Short preface:
The first version of JavaScript was completed in ten days in order to accommodate the Netscape Navigator 2.0 Beta release schedule.

Keep in mind that well structured languages usually take a little bit more than that.

What follows is a small piece of the aftermath of the rushed job.

NOTE: There is more madness in [destroy all software's wat video](https://www.destroyallsoftware.com/talks/wat), which I suggest watching before reading this post.

Explanations are at the end of the post.

## Truthy? Falsy? Schrödinger!
Let's start slow, with something that you have probably already heard of: truthy and falsy.

All ECMAScript values can be evaluated to boolean, and this is usually done in `if` statements or, when necessary, with the `!!` operator, that negates a value twice, casting it to boolean.

As a programmer that comes from C-like languages, one would expect `0`, `null` and `undefined` to be falsy, and that assumption holds.

The weirdness begins with empty values:
```js
!![]; // true
!!{}; // true
!!""; // false
```
And it gets worse:
```js
[] == true;     // false
!![] == true;   // true

{} == true;     // SyntaxError: "Unexpected token =="
true == {};     // false
false == {};    // also false...
!!{} == true;   // true

"" == [];       // true (please keep in mind the code block above)
```

Which brings us to ECMAScript infamous holy trinity. The following statements are all true:
```js
[] == 0;
"\t" == 0;
"0" == 0;

// but of course
[] != "\t";
"\t" != "0";
"0" != [];
```

---
## Who needs `int`s anyways?
When they designed ECMAScript they decided to get rid of `int`s altogether. Processors were starting to be really fast and optimizing algebra was not necessary anymore. What they forgot is that comparison is not reliable on doubles, but it is on `int`s.

A programmer coming from old, strongly typed languages, might expect `a + b + c === c + b + a` to always be true, but it is not reliably so if you only use doubles:
```js
a = 1/3;
b = 5/7;
c = 7/13;
a+b+c === c+b+a; // out: false
```

---
## What day is today?
In most languages support for dates is builtin, and ECMAScript is no exception.

Let's play with them a little bit, shall we?
```js
new Date() == new Date();      // false

// Okay, maybe the interpreter evaluates left to right:
new Date() < new Date();       // false

// Okay, maybe the interpreter evaluates right to left?
new Date() > new Date();       // false

// Mh... what about <=  and >= ?
new Date() <= new Date();      // true
new Date() >= new Date();      // true

// What the...
new Date() != new Date();      // true

// (╯°□°)╯︵ ┻━┻
```
Let's **stop** playing with them right now...

---
## `parseInt` flakiness

The lack of types in ECMAScript made it necessary to make some decisions on which types would prevail when operations would happen across different ones.

For example when you sum a string and a number, the number is cast to string and concatenated with the string:
```js
"1" + 1; // out: "11"
```
And this is mostly fine, it makes sense to use string as the dominant type: strings can represent more things than numbers.

But then you find out that you can multiply a string by a number, and that it is not a shorthand for `String.prototype.repeat` as it is in python, but implicitly casts the string to a number and executes the multiplication:
```js
" 2" * 4; // out: 8     <- NOTE THE LEADING SPACE
"a" * 4;  // out: NaN
```

But even this... Is mostly fine... I guess.

One of the consequences was the need to have explicit conversion helper functions to go back from string to number. It is true that it was possible to multiply by one to cast to number, but that made everything harder to read (more than usual) so they decided to provide two helper functions: `parseInt` and `parseFloat`.

And they behave **exactly as expected**:
```js
parseInt(0.001);        // out: 0
parseFloat(0.002);      // out: 0.002
parseInt(0.0000003);    // out: 3              <- ??
parseFloat(0.0000004);  // out: 0.0000004
```

---
## (Dys)functional approach
This is just amazing. In modern ECMAScript it is strongly advised to use the functional approach instead of the old-style imperative one.

For example, instead of doing something like a `for` loop over an array and have an accumulator for results, use `forEach` or `map` functions to closure the s`**`t out of your core logic and make it more readable.

Why do something like
```js
const a = ["2","4","6","8","10","12"];
const out = [];
for(const strnum of a) {
	out.push(parseInt(strnum));
}
console.log(out);
// out: [2, 4, 6, 8, 10, 12]
```

When you can compress it and hide the real complexity of the code, and in the meantime obtain something completely different as a result?
```js
["2","4","6","8","10","12"].map(parseInt);
// out: [2, NaN, NaN, NaN, 4, 7]
```

---
## Are you `in` for this?
ECMAScript comes with a nice `in` keyword, that allows coders to check if an element is in a collection:
```js
a = [1, 2, 3];
2 in a;     // true
4 in a;     // false
```

The `in` keyword also servers as an iterator over collections:
```js
let a = [0, 1, 2 ];
for (let i in a) {
	console.log(i);
}
```
prints
```
0
1
2
```
but also
```js
let a = [ 2, 1, 0 ];  // <- Note the order
for (let i in a) {
	console.log(i);
}
```
prints
```
0
1
2
```

---
## Collections
You might expect all Arrays in ECMAScript to have the `map` method. Thus you might also expect that since you can do:
```js
for ( let link of document.getElementsByTagName("a")) {
	console.log(link);
}
```

You can also do:
```js
document.getElementsByTagName("a").map(link => { console.log(link); });
```

But you can't, because that is not an Array, it is an `HTMLCollection`. Different Object, different collection, different rules.

---
## Scoping
If you come from any C-like language you might be used to [variable shadowing](https://en.wikipedia.org/wiki/Variable_shadowing).

Shadowing can be used to prevent accidental assignments to outer scope variables and to use less names in the code to keep it more readable.

If this is the case, you will surely guess the output of this code:
```js
var x = 1;
if (x === 1) {
  var x = 2;
  console.log(x);
}
console.log(x);
```
Which, of course, is
```
2
2
```

But there is more: variable declarations in ECMAScript are hoisted, this means that this is valid code, and prints `1`:
```js
x = 1;            // <- No undeclared variable error
console.log(x);
var x;
```

---
## Dictionary? Object? Both!
Let's say you want to count the words in a piece of text, so you write a small function like:

```js
function countwords(text) {
	let wordCount = {};
	for (let word of text.split(" ")){
		if (wordCount[word]) {
			wordCount[word] += 1;
		} else {
			wordCount[word] = 1;
		}
	}
	return wordCount;
}
countwords("This works and works again");
// out: {"This": 1, "works": 2, "and": 1, "again!": 1}
```
It always works, until it doesn't:
```js
countwords("I never implement a constructor when I use js!");
```
```JSON
{
	"I": 2,
	"a": 1,
	"constructor": "function Object() { [native code] }1",
	"implement": 1,
	"js!": 1,
	"never": 1,
	"use": 1,
	"when": 1,
}
```

---
## Try catch finally
This is not different from Java, so I guess it works as intended, but still sounds wrong:
```js
let riddle = function(){
	try {
		throw new Error("a");
	} catch (e) {
		throw e;
	} finally {
		return "a";
	}
}
riddle();   // out: "a"
```

No Error is thrown. It is just lost in the stack. [Here](https://blogtitle.github.io/considerations-on-error-handling/) are some explanations and considerations on error handling.

---
## `this` is Sparta!
Since ECMAScript (originally JavaScript) tries to be a scripting language that looks and feels similar to java, it comes with the `this` keyword.

Now, that keyword is so surprising and unreliable that there is a very long (741 words) stack overflow answer [here](https://stackoverflow.com/a/3127440) that shows some issues with it. I don't even want to try and summarize it.

The sole fact that there is a [3375 words long blog post](http://javascriptissexy.com/understand-javascripts-this-with-clarity-and-master-it/) that tries to make it simpler is telling a lot.

---
## With's end of wit
To be honest, the `with` statement is now deprecated, so this is mostly here for historical reasons.
```js
function madness(param) {
    with(param) {
        // what will param refer to?
        console.log("param: "+param)
    }
}
madness("this is an argument");           // out: param: this is an argument
madness({ name: "this is an object"});    // out: param: [object Object]
madness({ param: "this is a field" });    // out: param: this is a field
```
This was in the language, and still is, but is now discouraged. To put this in [Brendan Eich](https://en.wikipedia.org/wiki/Brendan_Eich) [words](https://twitter.com/brendaneich/status/68001466471817216): "with violates lexical scope, making program analysis (e.g. for security) hard to infeasible."

If you don't know who that is, he is co-founder of Mozilla and creator of JavaScript.

---
# Some explanations:
### The trinity:
This clearly happens for weird type casting.

* When casting to string an empty array it becomes empty string
* When casting to number all empty spaces are dropped, and all empty strings are zeroes

### Number comparison:
Floating point sums are not commutative due to [rounding](https://en.wikipedia.org/wiki/Floating-point_arithmetic#Representable_numbers,_conversion_and_rounding). The issue with ECMAScript is that it is not so simple to make sure your numbers never turn to doubles, while in all strongly typed languages you can just declare them as some kind of `int`.

### What day is today?
It is very well explained on [this stack overflow comment](https://stackoverflow.com/a/36926905)

### `parseInt` flakiness
This happens because `parseInt` expects a string. If a string is not provided the argument is _cast to string_ and then used. When a floating point number with more than 6 leading zeroes before the significant digits is converted to string, it is also converted to the exponential notation.

This turns 0.0000003 into "3e-7", but 0.002 into "0.002". Since `parseint` stops at the first non-digit character, it only reads the "3".

### (Dys)functional approach
`parseInt`, as many other functions, accepts more than one parameter, but only one is mandatory. The second parameter is intended to be _the base_ the first parameter is represented in, from 2 to 36 and defaults to 10.

When the `map` method is called, it passes three parameters (not one) the function. The tuple is: `(value, index, array)`. This means that the actual arguments passed to `parseInt` are:

```
("2", 0, a)     // 2 in base 0 is 2... I guess...
("4", 1, a)     // NaN: digit is bigger than base
("6", 2, a)     // NaN: digit is bigger than base
("8", 3, a)     // NaN: digit is bigger than base
("10", 4, a)    // 4
("12", 5, a)    // 7
```

`parseInt` ignores the array as it only accepts 2 arguments.

The correct code would be

```js
["2","4","6","8","10","12"].map(x => parseInt(x));

```

### Are you `in` for this?
This is because `in` returns the _keys_ of the given collection, while `of` returns the values.

### Scoping
The explanation can be found [on MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var#Description)
