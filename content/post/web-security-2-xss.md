---
title: "Rob'n Go security pearls: Cross Site Scripting (XSS)"
date: 2019-09-05T12:30:40+02:00
categories: ["Web security"]
tags: ["web","security","golang"]
authors: ["Rob"]
draft: true
---
> This post is part of my developer-friendly security series. You can find all other posts [here](https://blogtitle.github.io/categories/web-security/).

# Preface: authentication
In [the previous post](https://blogtitle.github.io/robn-go-security-pearls-fundamentals/) I briefly described how HTTP works and, as you might have noticed, it is a stateless protocol. Roughly every time you need to fetch a resource you have to issue a new request and if you need to access some restricted endpoint you need to **authenticate again**. As you can imagine this would cause some degradation in user experience if so the platform provides some ways to get around the problem.

Cookies are a built-in mechanism: whenever a server response contains a `Set-Cookie` header the browser will store the value in a so-called jar. From that moment on and until the cookie expires the browser will attach the cookie value to every request that is directed to that "endpoint" (more about this below).

When a user authenticates to the server the cookie is bound to their session and to their credentials on the server side, so that the user won't need to authenticate again.

Steps of a standard authentication process:

* User visits the website and might be issued a unique cookie;
* User logs into the website with their credentials;
* The server issues a new unique cookie and stores it in its database together with a reference to the user;
* Every time the user agent(usually a browser) requests a resource in the scope of the cookie it also sends the cookie along;
* When the server receives a requests it will lookup the cookie in the database and respond based on the user permissions.

The server might not use a database but might decide to sign the cookie somehow to recognize the user later. Moreover some applications might use authorization tokens instead of cookies. For the purpose of this post you can just assume that the JavaScript code running in a web application where the user is authenticated can do everything the user could. This is because cookies will be sent with every request and the other tokens can be retrieved by JavaScript at any time.

This means that **developers have to be very careful with which code they allow to execute** in the context of their applications.

# Enter the vulnerability
Most programs and web applications need to store and/or send back some data to the users. Modern web applications have hundreds of inputs and hundreds of outputs. As you might imagine sometimes developers get some escaping wrong and that is where things start going bad.

Let's make an example:

```html
<!-- This assumes you're using text/template -->
<div>
Hello I'm {{.Username}} and welcome to my shop!
</div>
```

If the username is `Rob` the output of the page will be

```html
<div>
Hello I'm Rob and welcome to my shop!
</div>
```
This works as intended. But what happens if the username is `<script>evilCode()</script>`?

When the browser receives the code, it can't figure out that the string above should be treated as text, so it executes it:

```html
<div>
Hello I'm <script>evilCode()</script> and welcome to my shop!
</div>
```
In this example the `evilCode()` function is invoked.

An attack scenario would be that an evil user signs up in this application then sends a link to its profile to the victim. When the victim visits the profile the name of the attacker is improperly interpolated in the page and evil code is executed. This will cause the application to **perform actions in the name of the victim without them knowing**.

# Server-Side XSS
The example above is an instance of server-side XSS because the interpolation happens on the server. To be more specific this is a case of stored XSS as it persists in the database of the application.

Some XSS might only trigger by interpolating data from the request into the response, and those are called reflected XSS. One example of reflected XSS in Go would be:
```go
func vulnerableHandler(w http.ResponseWriter, r *http.Request) {
  r.ParseForm()
  tok := r.FormValue("token")
  if !isValid(tok) {
    fmt.Fprintf(w, "Invalid token: %q", tok)
  }
  //...
}
```
This would reflect part of the input in the output without escaping, which causes XSS (you should never use `fmt` to write to an HTTP response).

In both reflected and stored XSS the culprit is a **lack of proper escaping**.

If you, like me, are a developer you **don't want to care** about all the possible escaping contexts you are putting user controlled data in, you just want things to work. As I like to think about this: stuff should be secure by default and not require the developers to care about it. Libraries should be close to impossible to use wrong.

If you are working with Go you are lucky. The `html/template` package does contextual auto-escaping. This means that when your templates are parsed the library detects in which context you are putting strings in and it will pick a chain of escaping functions to properly encode it.

If you are interpolating, for example, in this context:
```html
<script>
  var a = "{{.Data}}";
</script>
```
This will automatically be encoded in a way that an attacker could not break. This means that 2 encoders will be executed:

* JavaScript string: characters like `\` and `"` will be escaped like `\\` `\"` so that the attacker cannot "get out" of the JavaScript string context.
* HTML: characters like `<` will be encoded to `&lt;` so that if the attacker is called `</script>` it will not break the "script" context and won't be able to inject HTML in the sources of the page.

If the first escaping wouldn't have been performed a string like `"; evilCode(); var b = "` would have executed `evilCode`.

This might seem trivial but there are some complicated cases where escaping might not be intuitive, like inside `style` blocks or HTML attributes (where escaping depends on the attribute key). Currently the standard library supports 16 different escaping functions and it goes through 24 different possible valid states and contexts while parsing the templates. It is very well designed and has only very minor known issues. I advise against trying to do this by hand or re-implementing this logic yourself. For Go 1.13 as long as you don't interpolate data in [an html tag name](https://github.com/golang/go/issues/19669)(and why would you?) or [a template literal](https://github.com/golang/go/issues/9200)(you shouldn't use two templates together anyways) your app should be in a pretty good shape.

Moreover all widespread open-source libraries that I found in the wild for Go and Rust **do not** perform contextual auto-escaping so keep in mind that if you want to use a template that is not `html/template` **you are putting yourself at risk**. On this topic I can only suggest to stick to the standard library to stay safe.

> Note: in the near future Google is planning to open-source [another Go library](https://github.com/golang/go/issues/27926) that grants an even better level of protection against XSS. Stay tuned to be updated when it is released!

If you want to know more about some bypasses and advanced details about contextual auto-escaping [here](https://rawgit.com/mikesamuel/sanitized-jquery-templates/trunk/safetemplate.html#problem_definition) is your poison.

# Client-Side (DOM based)
If you think using the proper template will save you from XSS, you're out of luck. There is still a whole family of XSS that is waiting behind the corner to hit you and your users when you less expect it. The server might do all the escaping correctly depending on the context the strings are interpolated in but, then, the client-side code might just decide to execute arbitrary code.

The Document Object Model(DOM) is a way to programmatically access and modify a webpage. JavaScript can call special functions and methods that can, during or after page load, change some things.

Let's look at an example:

```html
<script>
var mydiv = document.getElementById("myDiv");
mydiv.innerHTML = window.location.hash;
</script>
```

If the current page URI ends with `#<img%20src=x%20onerror="alert(1)"/>` this code will execute `alert(1)` in the context of the page. The attack scenario in this case is that the attacker sends to the victim a link like:

`https://vulnerable.com/#<img%20src=x%20onerror="evilCode()"/>`

and if the user clicks on it the attacker executes `evilCode` in their session.

As you can see no server-side code was involved. This is true to the point where the "frame" part of a URI (the string after and including the first `#` character) **is not even sent to the server**.

As a developer you might want to protect against this kind of vulnerability and **you might expect** the web platform to provide standard APIs for escaping and some well-defined ways to disable all JavaScript APIs calls that might result in an XSS.

Well... It doesn't `¯\_(ツ)_/¯`.

Or at least it doesn't **YET**. Proper escaping can be done with libraries like [`DOMPurify`](https://github.com/cure53/DOMPurify) and it won't get in the platform for a while so no luck there, but a way to enforce a secure access to the DOM APIs like `innerHTML` is being worked on. The idea to address the issue is to add [Trusted Types](https://github.com/WICG/trusted-types) to browsers so that only controlled strings can be used to access the DOM.

This is not implemented in stable versions of browsers but it will get there fairly soon so we might eventually be able to address this mess. If you want to try them out in advance there is [a polyfill library](https://github.com/WICG/trusted-types#polyfill) ready for you to use!

# Mitigations
There are some features in the web platform that were built to help out. In the sad case in which some attacker-controlled code might end up in your HTML page there are some countermeasures that might still render the exploitation extremely hard or, sometimes, impossible. Those countermeasures help in both server-side XSS and DOM XSS, but they have limited benefit in the latter.

### Mitigation: Content Security Policy
Content Security Policy (CSP) allows developers to declare which scripts they trust so that everything else won't be executed by the browser. Let's see an example of HTTP HTML response protected with strict CSP:

```http
HTTP/1.1 200 OK
Content-Type: text/html;
Content-Security-Policy: script-src 'nonce-thisIsSecureRandom'
Content-Length: 156

<html>
<body>
<script nonce="thisIsSecureRandom"> 
// This gets executed.
</script>
<script>
// This does not get executed.
</script>
</body>
</html>
```
The idea is to use an HTTP header to communicate to the browser a secret that changes for every HTML response generated by the server. Only script that have the `nonce` attribute set to that secret will be executed. This means that the attacker will need to guess the random or somehow predict it in order to get code to execute. This makes exploitation significantly harder and sometimes impossible.

If you want to (and you should) adopt CSP on your application you can take a look at [Google official guide for it](https://csp.withgoogle.com/docs/strict-csp.html) which contains an actual policy Google uses in production (the one above is oversimplified and might cause problems with some browsers).

> Note: There currently is no library in Go to automatically add nonces and CSP but [I'm working on it](https://github.com/golang/go/issues/31107) and will update you when it is ready.

In case your application doesn't use JavaScript and you want to make sure it never will, not even by accident, you can set content security policy to the one described in the guide above and replace the `script-src` directive to be `'none'`. This renders XSS impossible to exploit.

`Content-Security-Policy: base-uri 'none'; object-src 'none'; script-src 'none';`

If you are going to use trusted types they will also be sent as part of your CSP.

### Mitigation: Cookies scope
Cookies should be scoped and restricted to only be sent when necessary. Moreover cookies should not be directly accessible by JavaScript. This can be achieved with the [`Domain and Path`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#Scope_of_cookies) and [`HttpOnly`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#Secure_and_HttpOnly_cookies) attributes. This provides little mitigation against XSS but it is very simple to deploy so it is still worth the effort.

# Recap
If you want to secure you application from XSS you should:

1. Use templating engines with contextual auto-escaping ([Go standard library](https://golang.org/pkg/html/template/) or [Google closure templates](https://github.com/google/closure-templates) do it);
1. Adopt [strict CSP](https://csp.withgoogle.com/docs/strict-csp.html);
1. Add [Trusted Types](https://github.com/WICG/trusted-types) to your CSP whenever possible (even polyfilling should work fine) or make sure to carefully review all uses of [dangerous DOM APIs](https://wicg.github.io/trusted-types/dist/spec/#injection-sinks);
1. Set your cookies as [`HttpOnly`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#Secure_and_HttpOnly_cookies) and [scope them](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#Scope_of_cookies) so that only the endpoints that need them receive them.

# Let's apply this to Go!
* Trusted Types are client side, so not much for Go to do, but we can work on the rest.

* To generate responses make sure you always write to an `http.ResponseWriter` with an HTML template and nothing else. It should be easy to catch this mistake in code review but it is also quite easy to write an analyzer for it. If you want to try and write such a tool here is the [doc](https://godoc.org/golang.org/x/tools/go/analysis) and [a talk](https://www.youtube.com/watch?v=HDJE-_s3x8Q) about it. If you end up writing it, please [share it](https://staticcheck.io)!

* Strict CSP and contextual auto-escaping can be done with something like the following proof of concept. Make sure you import `"html/template"` and `"crypto/rand"` and **not** `text/template` and `math/rand`.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"html/template"
	"log"
	"net/http"
)

const (
	cspKey = "csp-nonce"
	cspTpl = "object-src 'none'; script-src 'nonce-%s'; base-uri 'self'"
)

func protect(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		nonce := genNonce()
		w.Header().Set("Content-Security-Policy",
			fmt.Sprintf(cspTpl, nonce))
		ctx := context.WithValue(r.Context(), cspKey, nonce)
		h.ServeHTTP(w, r.WithContext(ctx))
	})
}

var tpl = template.Must(template.New("hello").Parse(`
<script nonce="{{.CSPNonce}}"> alert("It works!"); </script>
<script> alert("This doesnt!"); </script>`))

func main() {
	http.Handle("/", protect(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			tpl.Execute(w, map[string]interface{}{
				"CSPNonce": r.Context().Value(cspKey),
			})
		})))
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func genNonce() string {
	var b [20]byte
	if _, err := rand.Read(b[:]); err != nil {
		panic(err)
	}
	return base64.StdEncoding.EncodeToString(b[:])
}
```
Please note that here I'm decorating only a single handler but an entire `http.ServeMux` can be protected with CSP in this exact way.

Cookies attributes can be set together with cookies by using [the standard http package](https://golang.org/pkg/net/http/#Cookie).

This is all for today, stay tuned for the next chapter: Cross Site Request Forgery(CSRF).
