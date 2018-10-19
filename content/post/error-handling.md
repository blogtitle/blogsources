---
title: "Considerations on error handling"
date: 2018-08-10T12:34:41+02:00
categories: ["Golang", "Software engineering"]
tags: ["Funny"]
authors: ["Rob"]
---

## The impact
The first language I used to build something more than just school exercises was VisualBasic.Net. There are some good things and bad things in it, but I don't want to discuss that today. I switched to C# after that and Java and Python after it.

This is to say that basically up until three years ago I knew one and only one way to handle errors which is expressed in the following pseudocode:
```python
try:
	# dostuff
except: # or catch
	# handle problems
finally:
	# make sure I didn't forget anything
```

That's it: out of band error signaling (errors flow in a different channel than normal return values), with explicit specialized syntax.

I was only writing python and C# when I felt the need for something more powerful on the concurrency/parallelism side. Working with networking and concurrent, deeply parallel, code with python is just a hassle, and C# was way too verbose.

That is one of the reasons I started trying Go. And the impact was *awful*.

I loved the concurrency patterns and the `go` statement. But I stumbles on the lack of features, the lack of good UI libraries, the lack of constructors, the lack of proper package management. But what really pissed me off was the error handling:

```go
result, err := doStuff()
if err != nil {
	// handleProblems
}
```

My first reaction to this was: "Did they just forget about errors while designing the language?"

But then I discovered this: when you return a nil error, make sure you return `nil` and not a `nil` value of a `struct` implementing the `error` interface or [it will not be nil](https://golang.org/doc/faq#nil_error):
```go
type MyErr string

// Implement error
func (m *MyErr) Error() string {
	return string(*m)
}

func main() {
	var err error
	// This is a nil pointer
	var myerr *MyErr
	err = myerr
	if err != nil {
		// But not a nil interface, because it has a type
		fmt.Println("err")
		return
	}
	fmt.Println("not err")
}
```
```
out: "err"
```
To which my reaction was just a long list of swear words.

All those things that I didn't like didn't prevent me from using Go for some projects: I went on exploring the environment, the small details, I started appreciating channels and loving the `select` statement.

## Some months later

After a while, almost all clicked in place, and while the package manager was just not there, but some experiments solved most of my problems ([dep](https://github.com/golang/dep)). The only thing that really kept bugging me was the verbose error handling.

I wanted to understand what was behind it and what was the reasoning that made the language creators settle on it. The [errors](https://golang.org/src/errors/errors.go) package is ten (10) lines long cleaned of empty lines and comments and that is everything that the standard library provides.

Simple is nice, but too simple is scary.

## The reasoning

I listened to [a very nice talk](https://www.youtube.com/watch?v=lsBF58Q-DnY), read [an official blogpost](https://blog.golang.org/errors-are-values) and saw a couple of potential solutions: [pkg/errors](https://github.com/pkg/errors) [ErrGo](https://github.com/juju/errgo). Those addressed some issues but nothing really seemed to justify the verboseness and the complete lack of fine-grained error catching.

Then [Mat Ryer](https://twitter.com/matryer) gave a valid explanation to the verbosity in [his talk on idiomatic Go tricks](https://youtu.be/yeetIgNeIkc?t=257) by introducing to me the concept of "happy path" and "line of sight".
That made me realize how much easier it is to read Go code than Java-like code: my eyes don't have to jump up and down between a try block and a catch block to understand the flow, and I don't need to know the Exception inheritance hierarchy and taxonomy to understand which catch block is going to be selected.

That explained the verbosity, and even though I'd like to have a short form for `if err != nil`, I would now never go back to try-catch-finally statements. I like to have error handling code near to the calls that caused the error.

What that didn't explain, although, was the lack of granularity of error handling. All errors are equal and if you need to understand the underlying cause of an error you can't: you shouldn't rely on strings in messages as they may vary, and you cannot inspect an unexported type coming from another package.

The point is that Go doesn't like strong contracts: functions should accept interfaces whenever they can, packages should export interfaces to prevent users from using the `structs` wherever it makes sense to do so. And that is also true for errors: a package that exposes granularity for errors might become harder to maintain (potential breakages if errors are added/changed or the underlying system is updated) and more complex to use.
This made it so that some packages export helper functions to check errors, like the `os` package for [permissions](https://golang.org/pkg/os/#IsPermission), or even export the error values like [io.EOF](https://golang.org/pkg/io/#pkg-variables). A nice perk on checking errors with boolean operators instead of using a `catch` block is that error handling code can fallthroug from a more specific handler to a less specific one:
```go
file,err := os.Open("foo.txt")
switch {
	case os.IsPermission(err):
		log.Println(accessErrorMessage)
		// Explicit fallthroug
		fallthrough
	case err != nil:
		return err
}
// The powerful defer statement, way better/different than a `finally` block
defer file.Close()
```

After some time spent coding Go, that starts to feel natural and if a package does not expose an error, you get used to the fact that it means _that package cannot guarantee forward compatiblity on that error_ and that's it, because in Go people try to do their best not to break other people's code. Panics are handled inside the package and updates do not break APIs, this requires weaker contracts in some cases.

## The comeback

Now it has been three years of writing Go, and I use it for everything: big software or scripting. Every time a bash script might get longer than ten lines, I switch to go. I started thinking with values, and errors are no different values. I stick to the fact that the syntax might give some sugar for `if err != nil` but that's it. The problem now is the opposite, I realized how uselessly complex is the try-catch-finally statement.

Let's give some examples with the "Guess the output" game. One of my theories is that code should **never be surprising** and traditional error handling might do sometimes:

### Taxonomy

One of the remarkable things about errors in go is that they are values, and the language focuses on that rather than their inheritance and taxonomy. This makes very hard to have ambiguities such as the following one:
```Java
try {
	// Explicitly declared as Exception
	Exception e = new PermissionException();
	throw e;
} catch (PermissionException re){
	log("Make sure you have permission");
	// No fallthrough
} catch (Exception e) {
	return "b";
}
return "a";
```

What does this code return?

Luckily Java well behaves and enters the first catch block, and the code logs correctly and returns "a".

But it gets trickier.

### Finally

Early returns usually make code more readable: they reduce the need for indentation and cyclomatic complexity, and they make sure code is never accidentally executed. But Java has some issues with early returns and error handling code:

```Java
try{
	throw new Exception();
} catch (Exception e){
	return "a";
} finally {
	return "b";
}
```
This code returns "b" as the `finally` block is always executed after the `catch` blocks, even if they invoked a return statement.

This is a little bit weird, but after a while you can get used to it, I guess. The madness comes right now.

### Eventually
```Java
try {
	Exception e = new Exception();
	throw e;
} catch (Exception e) {
	throw new RuntimeException();
} finally {
	return "a";
}
```
What does this code do? Who catches the `RuntimeException`?

Short answer: no one. This code returns "a". The stack escalation caused by the `throw` statement in the catch block is stopped by the `finally` block. And there is *no way* to catch that exception anymore. It is just lost.

This means that there is no real equivalent of a `defer` statement. If there is a need to catch and filter some exceptions, but you still let some other ones escalate the stack, the finally block must not be used. This prevents Java-like programs to have something like:

```
try:
	something
catch a_particular_error:
	// let other errors bubble up
anyways:
	// execute this cleanup code
	// and DO NOT continue execution
	// but don't stop other errors that are bubbling up
```

which in Go exists as
```go
defer cleanup()
err := something()
swtich {
	case err == myErr:
		handle(err)
	case err != nil:
		return err
}
somethingElse()
```

## Conclusion
Go error handling is not perfect. The fact that external packages have been written to provide more is a giveaway of this. It is, although, a change. It is something different that tries to demistify errors, and that is leading somewhere more where code is more readable and less confusing.

Some things that might seem tempting when coming from other languages but that would **not** be nice to have are:

* The `finally` statement, as the examples above try to show.
* Error handling relying on inheritance/embedding: errors should be values, and their taxonomy should not interfere with the semantics of the handling code.

There is, on the other hand, a lot to learn. It would be nice to have:

* More concise syntax
* If err is not nil, the linter/compiler should warn when the user is trying to access the other values returned by the call anyways. Even if there are [exceptions](https://golang.org/pkg/bytes/#Buffer.ReadBytes) to this rule, it is almost always wrong to read return values on non-nil errors. (this confusion can't happen in languages with exceptions as extra return values are discarded)
* Errors should be explicitly ignored. Since Go is very pedantic when [assigned values are not used](https://github.com/golang/go/blob/c359d759a77fcec1457f3eb5c5d04fb74f47dad4/src/cmd/compile/internal/gc/walk.go#L52), function calls that return only an error should, in my opinion, require to be called with the underscore syntax ( `_ = myFunc()`). This would work as a warning that errors are being ignored for the readers. It is otherwise hard to spot such mistakes.
* An idiomatic way to have some granularity like the one offered by catch. I like that the os package tries to export helper functions, but there should be consistency across the standard library. I was once trying to distinguish between different TLS errors, but the `tls` package only exports [one error](https://golang.org/pkg/crypto/tls/#RecordHeaderError) which was not the one I needed.

Please feel free to share this and discuss it on [twitter](https://twitter.com/empijei).

