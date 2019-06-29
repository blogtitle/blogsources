---
title: "Use Go as web server"
date: 2019-04-12T06:44:54+02:00
categories: ["Go", "Security"]
tags: ["go", "web", "security", "golang"]
authors: ["Rob"]
draft: true
---

Most people are used to build a web application in some technology, and then use nginx or lighttpd or caddy as web server. There is nothing wrong with this, but today I'm going into a little bit of details on how to build a web server from scratch in Go.

If you wonder why would you do this instead of using something pre-built, here are my main reasons for it:

* You know and control what is running on your system
* Less code that you don't need
* Unlike nginx or lighttpd it is written in a memory safe language
* It's easy to deploy: you build it and push it where you need it
* It has no dependencies, which [is good](https://research.swtch.com/deps)
* Configuration languages allow you to configure things **up to a point** and then you have to stop or be driven mad. Code doesn't.

All the sections in this post try to show a simple and at least one advanced example for a configuration.

# The Server Multiplexer
As first web-related thing in our `main` we will need to create an [`http.ServeMux`](https://golang.org/pkg/net/http/#ServeMux). This will be the server multiplexer for everything we will do from now on.
```go
mux := http.NewServeMux()
```

Here are some functionalities that you can use your multiplexer for:

## Virtual host
It is very common to have more than one host running on the same web server, and this is quite easy to achieve with Go:
```go
mux.Handle("my.foo.org/", foo)
mux.Handle("my.bar.dev/", bar)
```

`foo` and `bar` need to implement the [`http.Handler`](https://golang.org/pkg/net/http/#Handler) interface, which just has the `ServeHTTP` method. I'll provide some examples for this.

Keep in mind that **a `ServeMux` is an `http.Handler`** and thus this pattern can be nested at will.

## Static files
Let's say that we are now configuring the handler `foo`, and we want it to serve some static files:
```go
foo := http.NewServeMux()
// This servers static files under /static/ looking for them in ./foo/static
foo.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("./foo/static"))))
```
This enables directory listing on that particular path, and takes care of preventing directory traversal. If you want to disable directory listing but only allow direct access to files you can change it to be this:
```go
fs := http.StripPrefix("/static/", http.FileServer(http.Dir("./foo/static")))
foo.HandleFunc("/static/", func(w http.ResponseWriter, r *http.Request) {
	if strings.HasSuffix(r.URL.Path, "/") {
		http.NotFound(w, r)
		return
	}
	fs.ServeHTTP(w, r)
})
```

This is probably sufficient for most use cases. Please note that the Go static file server has special cases for `index.html` files. If this behavior is not fit for you you might want to provide a custom `http.FileSystem` with the behavior you prefer. Here is an example on how to serve `index.html` if there is such a file in the given directory, and return an empty page instead:
```go
type customFileSystem struct { http.FileSystem }
func (c customFileSystem) Open(path string) (http.File, error) {
	log.Println(path)
	f, err := c.FileSystem.Open(path)
	if err != nil {
		return nil, err
	}
	s, err := f.Stat()
	if err != nil {
		return nil, err
	}
	if s.IsDir() {
		return c.FileSystem.Open(strings.TrimSuffix(path, "/") + "/index.html")
	}
	return f, nil
}

// In a function:
	foostatic := http.StripPrefix("/static/",
		http.FileServer(customFileSystem{
			http.Dir("./foo/static"),
		}))
	foo.HandleFunc("/static/", func(w http.ResponseWriter, r *http.Request) {
		if strings.HasSuffix(r.URL.Path, "/") {
			http.NotFound(w, r)
			return
		}
		foostatic.ServeHTTP(w, r)
	})
```

### Proxying to another host
Sometimes it might be needed to proxy some requests over to another host. To do this Go provides a [`reverse proxy`](https://golang.org/pkg/net/http/httputil/#ReverseProxy):
```go
foo.Handle("/proxy",http.StripPrefix("/proxy/",httputil.NewSingleHostReverseProxy("https://empijei.science"))
```
This will proxy requests for `my.foo.org/proxy/content` to `empijei.science/content`.

This approach has some limitations: for example this does not rewrite the host header and it could cause some bugs. If you want to go deeper you can use a reverse proxy Director manually:
```go
foo.Handle("/proxy",http.StripPrefix("/proxy", &httputil.ReverseProxy{
		Director: func(req *http.Request) {
			req.URL.Scheme = "https"
			req.URL.Host = "empijei.science"
			// Leave Path and Query parameters unchanged
			if _, ok := req.Header["User-Agent"]; !ok {
				// explicitly disable User-Agent so it's not set to default value
				req.Header.Set("User-Agent", "")
			}
			req.Header.Set("Host", "empijei.science")
		},
	})
```

> NOTE: [`httputil.ReverseProxy`](https://golang.org/pkg/net/http/httputil/#ReverseProxy) is **very** configurable, and allows to modify the response, set a buffer pool and an error log. If you want to go deeper you should probably take a look at the exposed API.

## Decorate handlers to add functionalities
### Static security headers
Adding static headers is common and useful, and in Go that is just a matter of knowing which ones you want:

```go
const defaultStaticHeaders = [...]struct {
	key, value string
}{
	{"Strict-Transport-Security", "max-age=31536000; includeSubDomains"},
	{"X-XSS-Protection", "1; mode=block"},
	{"X-Content-Type-Options", "nosniff"},
	{"X-Frame-Options", "SAMEORIGIN"},
	// If you want to know how to properly configure CSP you can visit TODO
	{"Content-Security-Policy", "object-src 'none'; base-uri 'self'; frame-ancestors 'self'"},
	{"Cross-Origin-Opener-Policy", "same-origin"},
	{"Cross-Origin-Resource-Policy", "same-origin"},
}

// protect returns a handler that will always set security headers and then call h.ServeHTTP.
func protect(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		hs := w.Header()
		for _, h := range defaultStaticHeaders {
			hs.Set(h.key, h.value)
		}
		h.ServeHTTP(w, r)
	})
}
```

If you know what you are doing you can also directly set the headers map entries with this, which is more efficient but requires you to get the header casing right.
```go
hs := w.Header()
for _, h := range defaultStaticHeaders {
	hs[h.key] = append(hs[h.key], h.value)
}
```

## Logging
At this point I think you got the gist of this, and you probably already know how to do this:
```go
func addLogging(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		myBeautifulRequestLogger(r)
		h.ServeHTTP(w, r)
		myBeautifulResponseHeaderLogger(w)
	})
}
```
I'd advise not to do response logging except for headers, as response writers might have many more methods than the ones specified by the `http.ResponseWriter` interface. This could affect performace.

If you really need to do it for debugging purposes you can always record the response with `httptest.ResponseRecorder` in tests or you can implement a simple one yourself:
```go
type simpleResponseRecorder struct {
	status int
	buf    *bytes.Buffer
	h      http.Header
}

func (s *simpleResponseRecorder) Header() http.Header {
	if s.h == nil {
		s.h = make(http.Header)
	}
	return s.h
}

// If you are not interested in saving writes but just statuses you can
// embed the http.ResponseWriter, forward the writes, and just log the
// status when it is set.
func (s *simpleResponseRecorder) Write(p []byte) (int, error) {
	if s.buf == nil {
		s.buf = &bytes.Buffer{}
	}
	return s.buf.Write(p)
}

func (s *simpleResponseRecorder) WriteHeader(statusCode int) {
	s.status = statusCode
}

func (s *simpleResponseRecorder) Respond(w http.ResponseWriter) {
	if s.status != 0 {
		w.WriteHeader(s.status)
	}
	if s.h != nil {
		wh := w.Header()
		for k, v := range s.h {
			wh[k] = v
		}
	}
	if s.buf != nil {
		s.buf.WriteTo(w)
	}
}

// You can use it this way:
func addLogging(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		myBeautifulRequestLogger(r)
		s := &simpleResponseRecorder{}
		h.ServeHTTP(s, r)
		myBeautifulResponseLogger(s)
		s.Respond(w)
	})
}
```

# The Server
// TODO proper config

## Enabling TLS
// TODO proper ciphers

TODO rendering directory templates (as bonus)
