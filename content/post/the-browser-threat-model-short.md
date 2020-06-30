---
title: "A Short Summary of The Browser Threat Model"
date: 2020-06-29T11:10:33+02:00
categories: ["Software engineering", "Security"]
tags: ["web security", "browser", "same origin policy"]
authors: ["Rob"]
draft: true
---

This is a short, simplified and not at all comprehensive post on the browser threat model. I decided to write this because I often time find myself explaining it to newcomers, so I decided to write it down. Some technical terms are simplified or replaced with more familiar and commonly used ones for ease of fruition.

Throughout all this read please keep in mind that attackers are usually after user **data** (stored inside or outside the browser) or **resources** (the machine the browser runs on).

# Outside of the browser
## Software
The Web Platform does not protect users from threats that come from the operating system and the programs running on the same machine the browser runs.

An example of this is that if a malicious program is executed by a user, everything inside the browser run by that user can be assumed to be compromised. This doesn't mean that the user data will be immediately or trivially stolen, but that the browser has no strong protection against an attacker that is positioned **inside** the user's machine. That is the operating system responsibility.

## Hardware
Hardware flaws like spectre and other micro-architectural leaks are part of the browser threat model and user agents are trying to protect users against attacks that rely on them, but they are not quite there yet.

Note: this means that browsers are working to rely on the underlying operating system to address them (like using multiple processes) but not that they are solving the issue themselves.

## Servers
The server a browser visits is assumed to be trusted and authoritative (this might also depend on the protocol, but I don't want to tangent too much). This means that if a server or its certificate is compromised the browser has no way to tell and all the information it has stored for the application hosted on that server can easily be retrieved by who controls the server.

# Within the browser
There are several security relevant grounds to be considered.

## Browser extensions
If a malicious or flawed extension is installed in the browser most of the protection it offers to the user is immediately lost. There are countless records of this and a big branch of modern malware acts via browser extensions.

Extensions are almost as trusted as the rest of the browser (hence the name "browser extension") so if the user installs one they are exposing themselves to a serious risk.

Keep in mind that browsers are "User Agents" and as such they will do what the user want and act in the user stead, so if a user wants to install an extension that will break all the security boundaries of the web platform, they are ultimately allowed to do so.

## User Interface
There are some components of the browser user interface that are of fundamental importance, so there should be **no way** for extensions or sites to change them.

Namely those are:
* The user settings menus
* The tab selection pane
* The URL bar (this is the most important one)

This substantially means that anything outside the region a webpage is displayed should enjoy a much higher level of protection and should be **clearly distinguishable**. This is one of the reasons themes and extensions cannot change the way the URL bar is displayed, or how browser settings are organized. This is also why *every time* a page is put in full-screen mode the browser shows a warning.

## Ownership
### Origins

The web platform has a builtin set of logical boundaries which are always enforced and that rely on the concept of "ownership".

User Agents assume that whatever is served under a certain origin trusts everything else that is served by that very same origin, without limitations.

For example the page served on `https://empijei.science:443/cooking/` has complete and total visibility and access to what is served on `https://empijei.science:443/about/`. (Note: this is *almost* correct, but I don't want to complicate things too much and I want to keep this post short).

The summary is that there is **no strong boundary between pages served "same-origin"** and for two pages to be considered as such they need to share:
* the same host (empijei.science)
* the same port (443)
* the same scheme (https)

This means that if you are hosting a web application the content you serve under your origin (the triplet scheme,host,port) should **all be trusted and under your control**. Moreover this also means that when you are importing a script from a [CDN](https://en.wikipedia.org/wiki/Content_delivery_network) you are trusting that entity to access all the data on your application. The grim consequence of this is that if a single page in your application or your CDN gets compromised you have to consider all of it to be compromised.

Everything that is served under a specific origin is somewhat isolated by the browser from other origins, depending on the relationship between the two interacting parties. Note that the browser can just protect from **some** attacks, but not all.

### Sites
If two pages are cross-origin, they can still be **same-site**. The definition of same-site is not as simple as the same-origin one. Two pages are same-site when they share **the same "effective top level domain plus one"** (eTLD+1). This means that for example `docs.google.com` is same-site with `mail.google.com`. Here `com` is a top level domain (and also an effective one, more on this later), `google` is the +1, so they are same-site.
A more complicated example is that `blogtitle.github.io` is not same-site with `empijei.github.io`. This depends on the fact that "`github.io`" is in [a list](https://publicsuffix.org/list/public_suffix_list.dat) that is embedded in every browser. That list is kept to try and collect all domains that might host content owned by different entities in a TLD+1 subdomain. In the `github.io` case `io` is a top-level domain (TLD) and normally everything under it would be owned by a single entity (e.g. `github.io` and its subdomains would be all part of a single ownership, which would be different from `filippo.io`). This is not the case, because every subdomain of `github.io` is owned by a different person: everyone can have their page and scripts hosted there, including the page you are reading right now. `github.io` is thus added to the list of "effective top level domains". As such, it behaves as "com" behaves for domains like `google.com` and `facebook.com`.

User Agents use the concept of same-site as a middle ground between being two completely untrusted, unrelated origins and a full trust.

The summary of the concept of same-site is that if two pages are same-site they might be cross-origin, but likely owned by the same entity and it is safe to assume they somewhat trust each other.

[Here](https://web.dev/same-site-same-origin/) you can find a longer and more detailed explanation on the concepts of origins and sites (and schemeful site).

Everything that is not same-site is assumed to need complete separation, and all interactions between two **cross-site web** applications _should_ be agreed upon by both parties.

"should" is the keyword in the line above.

The web grew organically and its security model was not designed in the beginning (it is not even perfectly defined now). This meant that when security was patched on top of the existing platform, it had to be done in a more or less **backward compatible way**. If you come from a software engineering background your "legacy senses" might be tingling, and you would be right. The web has an almost unlimited amount of [cruft](https://webaim.org/blog/user-agent-string-history/) that has been piling up for a little less than 30 years, and this is particularly evident when you look at it from the security perspective.

The most notable example of this is that **User Agents cannot by default isolate different origins from each other**, or most of the web would break. You would not even be able to follow most of the links in this page or see it with the fonts I intended it to have.

This, of course, means that web applications have to somehow protect themselves from cross-origin attacks that are not handled by the platform, which are a technical [buttload](https://www.google.com/search?q=define:buttload).

The issues might be due to [lack of isolation](https://en.wikipedia.org/wiki/Cross-site_request_forgery), [lack of means to prevent injection](https://en.wikipedia.org/wiki/Cross-site_scripting), [lack of restrictions granularity](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options) and generally [hard to parse formats](https://bugs.xdavidhu.me/google/2020/03/08/the-unexpected-google-wide-domain-check-bypass/).

The list goes on, and is way beyond the scope of this post.

# 

Recently browsers have started providing features to enforce stricter isolation mechanisms...

# Browser vulnerabilities
Browsers are complex pieces of software and the complexity of the platform for sure doesn't help. [Even parsing URLs is hard](https://www.youtube.com/watch?v=0uejy9aCNbI) and it is one of the core fundamental parts that the entire web builds on top. Most browsers have some sort of protection to make it so if a part of them is compromised, it is still hard to get to most of the user data or compromise
