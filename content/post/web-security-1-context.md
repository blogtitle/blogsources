---
title: "Rob'n Go security pearls: fundamentals"
date: 2019-08-21T02:52:26-07:00
categories: ["Web security"]
tags: ["web","security","golang"]
authors: ["Rob"]
draft: true
---
Welcome to this introductory series on web security. I'll use simple snippets and hands-on examples to introduce fundamentals on the topic.
Most server-side code will be in Go but you'll be able to understand it if you know any C-like language.

This fist post is about the foundamentals: URI, HTTP, TLS, HTML and escaping. If you are already familiar with those concepts please feel free to skip to the next post.

# It starts with a URI
Universal Resource Identifier(URI) is something you deal with every day, whether you know about them or not. URIs are strings that commonly look like "https://github.com/empijei". Don't be fooled by the apparent simplicity of these things, they are not simple and often times can trip up experts in the field.

URIs are composed by the following parts:
```
// [block] means that `block` is optional
[scheme:][//[userinfo@]host[:port]][/]path[?query][#fragment]
```

The most common scheme are `http` and `https`, but `javascript` and `mailto` are also broadly used.
This first part instruct the application that needs to use the URI on how to interpret the rest. It is not unusual to have custom schemes for specific applications, so that everything else in the URI is determined by that application and can use a custom format. One example for this is `smb` which allows on some operating systems to browse remote files on other machines.

Whatever comes after the scheme depends on it, so if it is omitted the client application (the browser from now on) needs to decide a default one when the user types a URI in the address bar.
Sometimes in web pages even the host is not specified, so the browser must provide a default host and potentially a prefix for the specified path to locate a resource.

We will focus on the dangerous consequences of allowing `javascript` and relative URIs in the chapter on Cross-Site Scripting(XSS).

# It is transported over HTTP
When the browser needs to retrieve a resource it usually performs an HTTP(HyperText Transfer Protocol) request to the server. Modern browsers all use HTTP 1.1 or above and the conversation usually looks like this:

Client connects to the server, and sends:
```http
GET /hello HTTP/1.1
Host: localhost

```

Server reads the request and responds:
```http
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 11

hello world
```
You can try this yourself by connecting on port 80 of any HTTP server with netcat or telnet and writing the request manually (hit return twice after done typing the host).

Let's unwrap what is going on here:

* `GET` is the HTTP **method** or **verb** we are using. There is [a wide range of possible ones](TODO link), the most common being OPTIONS, GET, POST and HEAD.
* `/hello` is the path we want. Usually this is taken from the URI as anything that comes after the host[:port] portion and before the `#` character.
* `HTTP/1.1` is the protocol we want to use to talk to the server. This implies that a "Host" header will follow.
* `Host: localhost` is a header. Headers are always in the form `Key: value`. In this case this means that on the machine we are connected to we want to talk to the host that is responding to the name "localhost". This allows multiple virtual hosts to be served by the same machine. We will focus on the dangerous consequences of having the Host header specified so openly in the chapter on DNS rebinding attacks.
* An empty line is needed to signal that headers are finished and the request body (if any) will follow. In this case we are using "GET" and we didn't specify a "Content-Length" header so the server starts responding as no body is expected.
Note: in HTTP lines are separated by "\r\n" and not just "\n". While some modern servers might accept single separators it is always better to use both, so our request will end with bytes `[0x0d 0x0a 0x0d 0x0a]` in this case.

The server then responds:

* `HTTP/1.1` means the server agreed on the protocol, we can proceed.
* `200 OK` is the status. This means everything went fine with our request. If we requested, for example, a resource that was not on the server we would have gotten `404 Not found`. The standard statuses are numerous and can be found [on wikipedia](TODO link).
* `Content-Type: text/plain; charset=utf-8` is a composed header that instructs the client on how to interpret the response body. In this case it says that it is just plain text and uses [UTF 8](TODO link) as encoding. We will see the dangers of this header or the lack thereof in the chapter on XSS.
* `Content-Length: 11` tells the client that the body is eleven bytes long.
* Empty line signals to the client that headers are over and the body will follow. We will see the harmful consequences of using such a simple separator mechanism in the chapter on http splitting.
* `hello world` is the response body.

There are also HTTP2 and QUIC in the party of protocols that are currently used, but you can assume they roughly behave like described above. The only difference is in speed and the ability for the server to push data to the client without the need for a pending request or open websocket.

# It is optionally (hopefully) transported securely on TLS
Most modern web services support secure connections over HTTPS. HTTPS is nothing else than the HTTP protocol over TLS(Transport Layer Security TODO check). This protocol secures the connection against data theft and data tampering, and authenticates the server to be the one we want to connect to. TLS can potentially provide mutual authentication, thus allowing the server to authenticate the client.

The TLS handshake is briefly summarized below. Please note this is not an exhaustive description of TLS and it is simplified to the point it is not precise.

* Client: Hello server, I'd like to connect securely, I support cipher suites A, B and C.
* The Server checks client suites against its own to find a common one.
* Server: Hello client, this is my certificate with my public key and I'd like to use cipher suite B.
* Client validates the server certificate:
  * the current date is within the validity range (certificate is neither expired nor not yet valid);
  * it is not revoked (this needs certificate revocation lists to be updated);
  * the signature is valid and emitted by an authority the client recognizes(this requires the client to have some trusted root certificates).
  * These steps can recursively climb up the certificate tree.
* The client then encrypts some random bytes with the server public key. Only who has the private key can then decrypt them.
* Client: here is an encrypted blob, let's use this as a key from now on.
* The server decrypts the key and starts communicating with the client by encrypting all traffic with the symmetric key exchanged in the previous point.

The cipher suite agreed in the first part determine some important algorithms, the most relevant are:
* how to validate the certificate
* how to encrypt the traffic
* how to check for message integrity

# And it is described by HTML
HyperText Markup Language(HTML) is a twisted, fuzzy derivation of XML that describes how web pages should look and behave. Over the years HTML evolved from a markup language that allowed hyperlinks to a fully blown style-markup-code descriptor.

Most modern web application use HTML for structuring the page, Cascading Style Sheets(CSS) to describe its styles and JavaScript(JS) to implement logic and behaviors. Those languages can be mixed together in a single HTML page. This gets to the point where also images can be transferred inline inside HTML, with Scalable Vector Graphics(SVG) and `data` URIs.

Here is an example:
```html
<html>
<!-- This is HTML -->
<head>
  <style>
    /* This is CSS */
  </style>
  <script>
    // This is JavaScript
  </script>
</head>
<body>
  <img src="data:image/png;base64,ThisIsAnImageInBase64=="
    onerror="/* This is JavaScript */"/>
  <p>
    This is text.
  </p>
  <a href="http://this-is-a-url/">This is text for the link</a>
  <a href="javascript:alert('This is JavaScript')">Run code</a>
  <svg>
    <!-- This is an image -->
    <sript>
      // This is JavaScript in an image :)
    </script>
  </svg>
  <div style="/* This is CSS */">
  </div>
</body>
</html>
```
Having data mixed with code is never a good idea. We'll see why in the chapter on XSS.

# Which requires escaping
As you probably noticed from the previous paragraph snippet, depending on teh context I was writing I had to use different comments delimiters. This is due to the fact that when HTML is processed there are several parsers at play.
For example when the HTML processor sees the `<style>` tag, it knows that all the content of that tag until `</style>` is for the CSS parser. The same is valid for the other contexts: the HTML processor will need to call into other parsers and the JavaScript engine to correctly deal with the page.

This fact has the interesting consequence that in JavaScript it is impossible to have the code `var scr = "</script>";` and it is necessary to do something like `var scr = "</scr" +"ipt>";` to prevent the HTML processor from switching context. We will see the proper way to do it in a few lines.

The main problem is that the HTML parser has no knowledge of the internal state of the JavaScript one, so it needs to be instructed somehow to know when it is time to resume processing tags and the JS block is finished.

The right way to tackle this problem is to not use HTML special characters in non-HTML blocks. All characters that might be relevant for the HTML processor need to be "escaped" somehow.

For example the symbol `<` will be encoded as `&lt;`. When the HTML processor sees that sequence it knows that it needs to decode it as a `<` and pass the decoded string to the JavaScript interpreter. this would turn `var scr = "</script>";` into `var scr = &quot;&lt;/script&gt;&quot;` which is not ambiguous anymore.

As you may imagine this requires some careful management by the programmer that is interpolating user data in a webpage, or the risk is that something that should be treated as data will be treated as code.

Stay tuned! Next post is on Cross-Site-Scripting(XSS) and will explain what happens when developers are not careful.

# Want to know more?
TODO links.
