Ideas:
* SQLi
* RCE
* MitM and session fixation (HSTS)
* Arbitrary file upload/download
* Denial of service and rate limiting
* ClickJacking
* Cross-Site Script inclusion
* Cross-Site search and other cross-site leaks
* Tabnabbing
* Encryption(transit+HSTS again,at rest,credentials,opaque PII to prevent logging and secure cookies)
* Authontorization, 2FA, weak logout and IDOR
* Account provisioning, what is a strong password and how to reset it
* Recap of security headers
* Information disclosure via headers or errors, fingerprinting and exposed pprof (never use default mux)
* Open redirect
* CORS


TODO add concept of origin and site:

### Origin
An origin can be thought as a "box" in the web. The browser will allow documents and servers in the same "box" to access each other in a lot of ways and with many channels. When, instead, documents are on different origins they are kept isolated by the browsers, and they can't freely communicate with each other's server.

"Origin" is thus one of the keys the browser uses to decide if an action should be allowed or not. An origin can be thought to be a triple of values: **protocol, domain and port**. For example the origin of this document is `("https", "blogtitle.github.io", 443)`. The port is not displayed by the browser because it's the default for https.

The following origins would not be same-origin with this document:

* `https://blogtitle.github.io:8443`
* `http://blogtitle.github.io`
* `https://notme.github.io`
* `https://stillnotme.blogtitle.github.io`

Two web pages that are not hosted on the same origin are called cross-origin.

### Site
When a page tries to do something cross-origin, like performing a request, the browser might want to understand if the destination endpoint would consider the current page a potential attacker or not. Using only origins would often times be too strict, so some boundaries have been relaxed.

"Site" is often times defined in many different ways, but for security-related topics the definition is very precise: two origins are same-site if they share the same **protocol** and **port** and their domains share the same **registrable domain**. A registrable domain is composed by one label plus a public suffix ([here](https://publicsuffix.org/list/public_suffix_list.dat) is a list of all the suffixes).

Some examples:

* `bar.com` and `foo.com` are cross-site: public suffix is `com` and the label is different;
* `a.bar.com` and `b.bar.com` are same-site: public suffix is `com` and the first label `bar` is the same;
* `blogtitle.github.io` and `notblogtitle.github.io` are cross-site: the public suffix is `github.io`, not just `io`, so labels are different.

This means that if a page hosted on `https://accounts.google.com` performs a request to `https://login.google.com` the browser would use slightly laxer restrictions than it would use if `https://evil.com` sent a request to `https://good.com`.

### What gets blocked
