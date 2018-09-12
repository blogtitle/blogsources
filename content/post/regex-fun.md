---
title: "Regexp Fun"
date: 2018-09-12T19:57:02+02:00
categories: ["Funny"]
tags: ["Funny"]
authors: ["Rob"]
---

# Preface
I love using regular expressions. They are such a powerful tool, they are complex, concise, complex, expressive, complex, broadly supported, complex and easy to write.

They are the perfect tool if you want to write a simple script and you need to elaborate text. Some downsides are that they are complex, surprising, and very inefficient compared with simpler string manipulations primitives (e.g `contains`, `hasPrefix`, `trim`, `replace`,...)

# Regular
The "reg" in "regexp" stands for [regular](https://en.wikipedia.org/wiki/Regular_language). This is very simple and linear, problem is that most regular expression engines do not respect this definition.

Perl-compatible regexps are a good example of non-regular language parsers, and that adds complexity.
This makes most regexps engines unfit and too slow/surprising to be used in production code, and I would likely reject any code that uses regexps instead of string manipulation functions (unless **strictly** necessary).

The following paragraphs are a brain-dump of some thoughts about regular expressions.
Never, for any reasons, use something like this in your code. **PLEASE**. If you somehow ended up on this page because you wanted to see how to parse a non-regular language using regexps, please refer to [this stackoverflow answer](https://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags/1732454#1732454) instead (which is a suggested read anyways).

# Why you should not use them
## Complexity
Most regexps engines use backtracking to match patterns. And that can lead to unpredictable behaviors.

I think that sooner or later any developer needs to trim a string, and they might be tempted by doing that with regexps. Let's take a look at the potential consequences.

A regexp that matches trailing white space in a string:
```
\s+$
```

This is so much more tempting than go to the end of the string and match backwards a set of characters and find the first non-empty space and slice the string, or something similar.

But it is also more dangerous. Let's say I feed the above regexp the following input:
```
this is a nasty                                                         example.
```
This string contains 57 spaces and then some letters. When the regexp engine tries to match it, these are the steps goes through:

1. save the cursor position
1. find a white space
2. start matching
3. advance (loop)
6. a non-empty space is found
	* if it is the end of the line
	 * MATCH!
	* else
	 * go back to the saved cursor, increment by one, repeat

In my example this will first match 57 characters, find that they don't end with a new line, but with an "e", backtrack one character, match 56 characters, find the same "e" as before, go back...

This will execute `(n * (n-1)) / 2` (1596) operations before giving up. As you can imagine, this is not something you want to have deployed in production. [Here](http://stackstatus.net/post/147710624694/outage-postmortem-july-20-2016) you can read an interesting post-mortem analysis on this behavior.

For more like this exploding behavior take a look [at this video](https://vimeo.com/112065252).

The detail that should worry you the most is that the previous example is a 4 characters long regexp that is trying to match a **regular language**. Imagine the consequences you could get with a much longer and more complex regexp.

NOTE: the golang builtin regexp engine does not exhibit this kind of explosive behavior, but this is not a good reason to use regexps everywhere.

## Surprises
Consider the following expression to match some Java v1 keywords (not all are included for brevity):
```
(for|
boolean|
if|
double|
implements|
protected|
byte|
else|
import|
int|
return|
extends|
final|
char|
short|
interface|
void|
class|
float)
```

If you try to use it, you will find out that the word "interface" is never matched by the capturing group.

Before continuing, please stop for a second to think how you would debug it, because this is not a trivial task to accomplish, especially in a complex piece of software that is exhibiting an unwanted behavior. The _last_ thing you are going to blame is a perfectly good looking regexp.

The bug here is that having "int" before "interface" makes most regexp engines match "int" and accept that as a good match, without looking further to see if "interface" can be recognized. This means that sets `(...|...)` should always be in **reverse alphabetical order** to behave as expected.

Simple, right?

# Now let's have some fun
I hope I have proven my point and made you think twice before using regexps in your code the next time you think about it.

That said, let's have some fun. The following regexps are the creation of an afternoon spent with [Anna](/authors/anna/) playing a [CTF](https://ctftime.org/ctf-wtf/) a couple of years ago.

If anything that follows leaves you puzzled, please visit [regex101](https://regex101.com/) that will probably explain better than me what is going on in the regexp. If the regexp doesn't run it's probably because I'm using some arcane (perl|ruby)-specific syntax, in that case please use [rubular](http://rubular.com/).

## Fun, part 1: push-down automata
Regular expressions, as said before, should only be able to match regular languages, but this is clearly not the case for PCRE (Perl-compatible regular expressions).

Let's start small: the language `a^nb^n`.

Some examples of strings in the language:
```
ab

aaabbb
aaaabbbb
```

This should only be matched by [push-down automata](https://en.wikipedia.org/wiki/Pushdown_automaton), as it requires a stack to memorize the amount of "a" that have been read, but we can easily write a PCRE to do so, leaving "[regular](https://en.wikipedia.org/wiki/Finite-state_machine)" or "finite state" matchers in the dust.

The regexp:
```
^(a\g<1>?b)$
```
This matches a group made of an "a", a "b" and optionally itself in the middle, which could only be another "ab" recursive sequence or nothing, and so on.

![more power](/xkcd/more_power.png)

## Fun, part 11: counting
Okay, let's push it further. The new language is now `x^p` where p is prime.

Some strings in the language:
```
xx
xxx
xxxxx
xxxxxxx
```

In case you are wondering, [1 is not prime](https://en.wikipedia.org/wiki/Prime_number#Definition_and_examples).

The regexp:
```
^(?!(xx+)\1+$)xx+$
```

Easy, right? We match any non-zero number of "x" with `xx+`, and we then assert with a negative lookahead that it is not a multiple of any number `(?!(xx+)\1+$)`. This can be put it easier in this way:
```
 xx+   : any number
(xx+)  : capture any number
(xx+)\1: capture any number, any exact number of times (m % n == 0)
(?!...): assert the following will not match
```
![more power](/xkcd/more_power.png)

## Fun, part 111: the interview question
The next time you are asked how to find if a word is palindrome, you can show off your regexp kung-fu and get a strong rejection from the company you are interviewing for, but have fun in the meantime.

The definition: a palindrome is a string that equals the reversed version of itself.

Some strings in the language:
```

a
aa
aba
abba
abcba
```

The regexp:
```
^((.)\g<1>\k<2+0>|.?)$
```

This is starting to be more complex, but bear with me for a moment.

Here we have a group `()` with an "or" `|`. The second case is an optional `?` single character, and that is the easy part. The first case, on the other hand, is kind of more complex. Let's break it down:

```
Match a character and save it. This group is automatically named with a number.
(.)

Matches the previous subexpression named "1". This re-executes the group, rather
than matching the same matched text. This refers to the outmost matching group.
\g<1>

Match the text matched by the group named 0+2, which in this case is (.)
\k<2+0>

Match a group that can be 0 or one character, or that can be generated in the
following way: <a letter>another instance of the group<the same letter>
((.)\g<1>\k<2+0>|.?)
```

[Reference for ruby regexp syntax (aka madness)](https://ruby-doc.org/core-2.1.1/Regexp.html).

![more power](/xkcd/more_power.png)

## Fun, part 1111: context sensitive grammar
This is the last snippet as I can feel my sanity slipping away.

If you got to this point, thank you, this requires an insanely long attention span, but beware: you still have to see the worst.

Language: `a^nb^nc^n` with `n>0` (let's get rid of the empty string for once).

Some strings in the language:
```
abc
aaabbbccc
```

Same number of "a", "b" and "c".

The regexp:
```
^(?=(a\g<1>?b)c)a+(b\g<2>?c)$
```


Let's break it down:
```
Lookahead, what comes after it must match what is specified by "..."
(?=...)

Match "a"<optionally itself>"b" and at least one "c". For more details, see
"Fun, part 1".
(a\g<1>?b)c

Match any non-zero number of "a"
a+

Match "b"<optionally itself>"c".
(b\g<2>?c)

Match any number of "a" followed by an amount of "b" equal to the same
amount of "c". The whole string must also match "a" followed by the same
amount of "b" and at least one "c".
^(?=(a\g<1>?b)c)a+(b\g<2>?c)$
```

That's it. Easy, right?

Thanks for reading and remember: don't use regexps in your code.
