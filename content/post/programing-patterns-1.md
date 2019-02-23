---
title: "Some useful patterns"
date: 2019-02-19T08:37:12+01:00
categories: ["Software engineering", "Go"]
tags: ["golang", "programming patterns"]
authors: ["Rob"]
---

Coming from a VB.Net, Java, C# and Python background, when I first started using Go I was very annoyed by the lack of some patterns on a language level. It took me a while using the language to find out those patterns can be easily expressed.

Here is a collection of some common patterns and the best way I've found to express them.

# Decorators
This feature is extensively used in most object oriented languages, and consists in somehow taking a function or method and enrich it with other effects or properties.

If you come from python you're probably familiar with something like
```py
@login_required
@app.route('/private')
def get_secret():
	# code here
```

Or something equivalent in C#
```cs
[Authorize(Roles = "User")]
public class SecretManager: Controller
{
	[Route("/private")]
	public ActionResult GetSecret()
	{
		// Code here
	}
}
```

These say that you need to be authenticated in order to access some data under `/private`.

There are several issues with this kind of programming practice that I find rather dangerous and annoying:

* It's easy to forget annotations, or misplace them
* In order to understand if the code is correct, you need to know an additional feature of the language (e.g. is order of annotations relevant?)
* It is complicated to find where those annotations are defined
* Control flow is hidden in glorified comments

The way you'd do this in Go is
```go
Authenticate(http.HandlerFunc(w http.ResponseWriter,r *http.Response){
 // Code here
})
```
A simple implementation for `Authenticate` would be:
```go
func Authenticate(h http.Handler) http.Handler {
	return http.HandlerFunc(w http.ResponseWriter,r *http.Response){
		if !isAuth(r) {
			w.WriteHeader(http.StatusForbidden)
			w.Write(forbiddenMsg)
			return
		}
		h.ServeHTTP(w, r)
	}
}
```
Moreover this pattern allows gophers to decorate entire types, not just functions. One could apply something similar for `(http.ServeMux).ServeHTTP`. This could be used to add default headers as in the following example:
```go
var securityHeaders = map[string]string{
	"Strict-Transport-Security": "max-age=31536000; includeSubDomains",
	"X-XSS-Protection":          "1; mode=block",
	"X-Frame-Options":           "SAMEORIGIN",
	"X-Content-Type-Options":    "nosniff",
}

type secureMux http.ServeMux

func (s *secureMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	for key, value := range securityHeaders {
		w.Header().Set(key, value)
	}
	s.ServeMux.ServeHTTP(w, r)
}

func setup() {
	// Cast the new ServeMux to a secureMux.
	serveMux := secureMux(http.NewServeMux())

	// Register handlers.
	serveMux.Handle("/private", Authenticate(myAuthHandler))
	serveMux.Handle("/settings", Authenticate(myAuthHandler))
	serveMux.Handle("/", greetingsHandler)

	// Use serveMux
	srv := &http.Server{Handler: serveMux /* more configs here */}
	if err := srv.ListenAndServe(); err != nil {
		log.Println(err)
	}
}
```
In this case spotting a missing call to `Authenticate` in the initialization phase is quite easy: all handlers are set up in subcessive calls in a single function, not where the handlers are defined. All routes are in one place and you can't forget some endpoints open.

# Singleton
Singletons are a common way to express "this thing exists only in one point of my program". There can be lazy or non lazy ones, depending on when initialization of the value happen. Non lazy singletons can be expressed as global variables, and they can be initialized during `init`  or directly in declaration:

```go
var myOnlyInstance *myType
func init() {
	// Prepare things that are needed here

	myOnlyInstance = newMyType()
}
```
or, better if no previous initialization is needed:
```go
var myOnlyInstance = newMyType()
```

This is trivial, and similar across most languages that I've used. The only big difference is that in Java/C# the value would need to be the static member of a class.

Go makes sure that `init` functions are done executing before running the `main` function, so this ensures all values have been initialized before using them.

Let's talk about the lazy ones now. In Java it may seem trivial to do it:
```java
public class MyType {
	static MyType single;
	public static MyType getSingle() {
		if (single == null) {
			single = new MyType();
		}
		return single;
	}
}
```
I've seen this pattern hundreds of times, but there is a problem in it: there is a race condition, a nasty one.
If multiple concurrent calls to `getSingle` happen, single will be initialized multiple times, breaking the constraint of it being a singleton. To do it right a `synchronized` keyword needs to be added, but it is quite heavy on a performance standpoint.

The same pattern can be expressed in Go with `sync.Once`:
```go
// The zero value for Once is ready to use
var oSingle sync.Once
var single *myType

func getSingle() *myType {
	oSingle.Do(func(){ single = newMyType() })
	return single
}
```
This ensures three main good things:

1. One and exactly one call to `Do` invokes `newMyType()`
1. Concurrent calls while `newMyType` runs block until the first one has returned
1. Calls after initialization have a very efficient fast path

For those interested about performance [a recent change](https://go-review.googlesource.com/c/go/+/156362/) made calls to the fast path of `Do` take about ~0.5ns.

A derivation of this pattern and the decorator one also comes in handy if you need to turn a non-race-safe API into one that is safe to call multiple times concurrently.

For example, [it is not allowed to call `Wait` multiple times on the same `exec.Cmd`](https://github.com/golang/go/issues/28461) so you can wrap it up like this:

```go
type multiWaitableCmd struct {
	exec.Cmd // Promote all Cmd methods
	o   sync.Once
	err error
}

// Wait decorates `(*exec.Cmd).Wait` with a `(*sync.Once).Do()` call
func (mwc *multiWaitableCmd) Wait() error {
	mwc.o.Do(func() { mwc.err = mwc.Cmd.Wait() })
	return mwc.err
}
```

# Static members
This section is just here because in some very rare occasions you need to have all instances of a certain struct share some values. One example of this I recently stumbled across was for debugging purposes: an interface I was implementing had a `Kind() string` method on it that was used for logging.

This can be done very easily without cluttering package namespace.
```go
type myImpl struct{}

func (*myImpl) Kind() string {
	return "Best implementation"
}
```
Note that `Kind()` has a pointer receiver, but does not name the variable, so it is clear it doesn't use it.

# Semaphores
Since I've always been very interested concurrency patterns, I was surprised to see that Go had no standard semaphores.

It was relatively quick to find out how to emulate them with channels:

```go
// Semaphore can be created with `make(Semaphore, size)`
type Semaphore chan struct{}

// Lock uses channels send operations to emulate an "acquire".
func (s Semaphore) Lock()   { s <- struct{}{} }
// Lock uses channels receive operations to emulate a "release".
func (s Semaphore) Unlock() { <-s }
```
Note that `Semaphore` implements `sync.Locker`. This code works because trying to send over a full channel will block until some other goroutine calls `Unlock()` and reads from the channel. If this is instantiated with size 1 it will have the same semantics of a `Mutex`.

A more efficient and weighted implementation can also be found in [the exprimental `sync` package](https://godoc.org/golang.org/x/sync/semaphore).

# Errgroups
Sometimes you want to spawn multiple goroutines and have them work in parallel, but when something goes bad or you don't need the output anymore, you would also like to cancel the entire process.

Achieving this with `sync.Waitgroup` and `context.Context` is doable but it requires a lot of boilerplate, so here I'd suggest using [`errgroup.Errgroup`](https://godoc.org/golang.org/x/sync/errgroup).

Example usage:
```go
import "golang.org/x/sync/errgroup"
```
```go
	eg, ctx := errgroup.WithContext(context.TODO())
	for _, w := range work {
			w := w
			g.Go(func() error {
				// Do something with w and
				// listen for ctx cancellation
			})
	}
	// If any of the goroutines returns an error ctx will be
	// canceled and err will be non-nil.
	if err := g.Wait(); err != nil {
		return err
	}
```
