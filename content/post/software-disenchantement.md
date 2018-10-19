---
title: "Software Disenchantement, AKA Oh $DEITY why is it so bad?"
date: 2018-09-24T18:07:11+02:00
categories: ["Software engineering"]
tags: ["Rant", "Funny"]
authors: ["Rob"]
draft: true
---

# Preface

The following is is a response to [this post](http://tonsky.me/blog/disenchantment/) but beware, it is just makes things worse.

I am also a security engineer, not just a software engineer, so I will add what I know best to the problems listed there.

DISCLAIMER: the following does not refer to non-public code of my current employer as I haven't seen enough and even if I did I couldn't speak about it, this refers to public code or code I've seen while working as a penetration tester in the past years.

That said, let the rant begin.

# Window management
This topic is not treated by the original post, so I'll try to cover it a little here, before talking about security.

I've been a Linux user for a while now, and on Linux you at least **get a choice** about window managers.

If you don't know what a window manager is, it is the piece of software responsible of (mainly) moving, resizing, pinning, snapping, reducing and restoring your windows. It is the code you probably interact with the most as a user. All the events that you fire from your keyboard and mouse get sent to the proper window in the right place thanks to your window manager.

When windows got "Areo Snap", which happened with Windows 7, it was a revolution... The problem is that it came several years after basically all linux WMs had it, and it came with transparency, animations, wobbly effects and alike, which made it so heavy I had to use the fallback option and change some registry keys to make it work on my laptop.

I just wanted to resize windows to half the screen when I draged them to one of the sides. Please note that this still doesn't work well in Windows 10 (You can't snap to the bottom/top half of the screen) and is still not present in MacOS. The most astonishing thing is that those window managers take up to 100 MB to do basically nothing. It's just fancy animations with zero provided additional functionality.

Linux is no exception to this trend of just making everything huge. Gnome became unbearable when it switched to version 3, in fact importing the Firefox renderer and JS engine to draw stuff (yes, you read it right) and KDE basically takes up all your resources and you better use a lightweight text editor in it or your transitions will start to lag.

Moreover, animations are everywhere and those are SLOW. I don't want to wait for an animation to end in order to start doing what I already wanted to do.

On a security standpoint window managers just don't consider it. When a pop-up is displayed it is able to get in front of all windows, and if the user was pressing "Enter" the default action for the pop-up is selected, making it impossible to recover what you just agreed to. And this was also true for administrative pop-ups for a long time.

# Fonts
Let's now talk about security implications of complexity, starting from the very basics: fonts.

Displaying text on a screen is one of the first things that terminals did at the very beginning of the history of personal computers, so we should now be good at it, right?

Not really.

Font rendering is a voodoo kind of dark magic with rules known to few people and that are mastered by no one.

The rationale for that is accessibility. We want to be more inclusive and less west-centric in modern software, so it makes perfect sense to have an advanced rendering engine for fonts.

The mad complexity required to draw some text on the screen doesn't just help supporting non-latin languages, but it also allows to create some very nice fonts, with enhancements for specific tasks. There are fonts with more space between lines to [help dyslexic people read faster](https://www.opendyslexic.org/), and even the original author of "Software disenchantment" created [a font with ligatures for coders](https://github.com/tonsky/FiraCode).

I think it is now time to talk a little about the **SMALL** security consequences of having the rendering engine being just a huge patch on top of an originally small code, written in an unsafe language.

The vulnerabilities found on only libtiff, one of the most imported libraries in the world for font rendering (used for example by Google Chrome), range from denial of service to remote code execution, the last one being a pretty common one.

TODO image

So even opening some text in the modern very complex ecosystem might lead a remote attacker to execute code on your machine.

If you still don't believe me look at what a simple, valid character (combining `~` accent) can do if repeated too many times, and how different software renders it differently:

TODO screenshot of my twitter, telegram, link to twitter on telegram, whatsapp, screen of daniel message on twitter.

But fonts are easy, let's switch to something slightly more complex, let's ad images and styles.

# PDFs
The original post says

> Bluetooth? Spec is so complex that devices won’t talk to each other and periodic resets are the best way to go.

If you think the Bluetooth spec is too complex, let's talk about PDFs for a second.

The [version 1.7 of the PDF reference](https://www.adobe.com/content/dam/acom/en/devnet/acrobat/pdfs/pdf_reference_1-7.pdf) (which is a PDF), published in 2006, has more than 1300 pages in it. It is extremely complex, and if you think that those 1300 pages should at least specify a very strict standard to avoid incidents like fonts did, oh are you off track...

Page 60 of this [(PoC||GTFO) issue](https://www.alchemistowl.org/pocorgtfo/pocorgtfo15.pdf) illustrates how a PDF can also be a git repository containing its own LaTeX source and a copy of itself. WHAT COULD POSSIBLY GO WRONG?

Not much, except even the Wikipedia page for PDFs has [a section dedicated to viruses and exploits](https://en.wikipedia.org/wiki/PDF#Viruses_and_exploits).

# Coding
This is a section on which most of you will probably not agree, so if you have strong feeling about your favorite language you should skip ahead.

* C: does not give you anything, you have to implement everything yourself, and it is unsafe, so that usually ends up in vulnerabilities all over the codebase.

* CPP: another unsafe language, but this one is also way more complex than C, because 4 different kinds of constructors (default, copy, move and parametrized) with god knows how many different syntaxes and coexisting references and pointers and smart pointers with relative arithmetic and rules seemed like a good idea.

* Java: astonishingly complex language too, luckily it is safe, less luckily its VM is [not so safe
](https://www.cvedetails.com/product/19117/Oracle-JRE.html?vendor_id=93)
TODO image JRE CVE

* Python: safe language again, but so inefficient that it has to rely on completely broken libraries written in C to at least be comparable to other languages. And let's not talk about the Python2/3 madness, virtualenv and pip.

* Ruby: basically the worse version of python. Very inefficient, relies on stuff written in C, the package manager is a never-working piece of unupdatable madness, and the syntax is so complex to deserve [a single 11540 lines long yacc file](https://github.com/ruby/ruby/blob/d325d74174c3ffffa0d75e248897cbb1aae93407/parse.y)

* Rust: reasonably safe language, but in order to get anything done you need to fight against the compiler over and over again. This causes "unsafe" code to be easier to write than "safe" code, which is a pretty bad anti-pattern in a language design. Moreover this is a clear example of over-optimization. The standard library, in order to be more efficient, uses `unsafe` extensively, causing vulnerabilities to surface every now and then.

TODO quote the two rust vulns.

* PHP: this is just a fractal of bad design, and the rationale about it is left as an exercise for the reader. I will probably soon or later write a post on this too.

* Javascript: I have a several comments on this. 
 * First:
TODO: spiderman laughing image 
 * Second: javascript doesn't even have the concept of a standard library, creating humongous dependencies for [even the most basic things](https://www.theregister.co.uk/2016/03/23/npm_left_pad_chaos/), causing a shower of security issues, creating the need [for a dedicated tool to keep track on them](https://docs.npmjs.com/cli/audit).
TODO node_modules image
 * The rest: I have [a dedicated post for it](TODO)

# HTTPS
Https is a protocol that is used to allow (mostly) your browser to communicate (mostly) securely with (mostly) all servers on the internet.

Let's take a look at what happens when you go YouTube to see a video of a kitten sneezing. I will just talk about the protocols and formats involved and their implementations, on a very high-level basis, the web platform will be discussed in the next chapter.

* DNS resolution: this step will have to go through either IPv4 or IPv6 to convert "youtube.com" into an IP address, this has been proven to be prone to vulnerabilities as it allows random people on the internet to send data to your computer that will be parsed, leading to potential code execution. (TODO systemd resolve RCE)
* TCP connection: the TCP protocol is pretty complex, and has several times lead to vulnerabilities, but should now have fairly secure implementations.
* SSL/TLS (hopefully the second): the TLS protocol is so complex and articulated that would probably require several tens of pages just to get started with it, and the most popular implementation of it is a broken piece of unmaintainable code. (TODO link openssl, get CVEdetails)
* HTTP parsing: this is a scaring, complex, old, text-based protocol that has been proven to be hard to implement (apache server/nginx vulns), and I'm not even considering the version 2 of it.
* HTML parsing: I believe HTML5 is the most complex language ever invented by mankind and so it was bound to lead to countless vulnerabilities even on the server side (cloudbleed TODO). The overall damages caused by the complexity of HTML is unknown, impossible to list here and ever-growing.
* Video/Audio decoding: I don't think I need to say that this is also very complicated, and that there are libraries like ffmpeg, on which chrome depends, that caused a long list of remote code execution vulnerabilities (TODO cve wrongly assigned)

Just

To

Watch

This

TODO sneezing kitten gif

# The web platform
I've spent a handful of years on this topic, so let's light some fires.

TODO gif scrocchiare collo

The web platform complexity is just unfathomable. I believe there is just a handful of people in the world that really understand every niche of it.

We can break the web platform down to 5 overly complex and dangerous components:

#### CSS
It was created to just help web programmers separate styles from the document structure, but it ended up becoming a Turing complete language that allows attackers to exfiltrate sensitive data via both direct use of features and advanced side channels.

#### JavaScript recognizing engine
JavaScript can be mixed with HTML exactly as CSS, it can appear inside the page, both in tags body and attribute values, as shown below.

```html
<img src=x onerror="alert('executing js')"/>

<script>
alert('executing this too')
</script>
```
Depending on the context it appears in, it needs different escaping/encoding/enwhatever and still can cause issues even if properly escaped, because browsers will do whatever is in their power to make sense of the broken pieces of HTML they receive from some outdated PHP or COBOL code that runs in the backend of your bank.

TODO Talk about security headers to prevent this

#### HTML engine
HTML is complex and very pathological inputs might put parsers in states that were not foreseen by coders, leading to memory corruption and potentially code execution.

TODO expand

#### JavaScript frameworks
Angular-polymer bad for security (CSP) and layers of ever more powerful metalanguages (angular lang)

TODO talk about how they shat on security headers

#### Renderer
Domato

#### JavaScript madness
I got suggested to use protobufs to make an extension communicate with a panel.

Js.js

Typescript

# About Electron and Apache Cordova
To quote the original post:

> Your desktop todo app is probably written in Electron and thus has userland driver for Xbox 360 controller in it, can render 3d graphics and play audio and take photos with your web camera.

Yeah, but it is much, much worse than that on a security standpoint

Electron XSS → RCE
Cordova Same Origin Policy disabled by default
ffmpeg, fonts etc

# Security is a compromise
Cost of performance and usability vs security, talk about spectre and meltdown cost, Security headers costs, rust design costs, a GC cost, etc.

# Complexity → Not secure
Restrict the unsafe/dangerous stuff to the minimum, the points of failures to just the bare necessary and review that

# Maintenance costs and software makers short sight
Rant s'more about python, JavaScript and PHP?

# Some software design suggestions

# Some solutions I adopt
I will create a more detailed post on how I have my workflow configured, but here is a summary of my current solutions:

Awesome, compton, vim, unix-like stuff, arch for lightweight and updates, Golang, hugo, put electron apps in the browser, use cables whenever I can.

FOLLOW THE UNIX STYLE, AND STAY IN THE CONSOLE.

About the rest, you are screwed.

# Post scriptum
Windows 95 was small in size indeed, but let's consider that plug'n play wasn't there and you could take it down with a malformed TCP packet.

Security adds complexity (cite the csp draft) and size of the software (ACL are complex but you WANT them to be there) and it is usually sacrificed because time-to-marked is too long, this UI is faster to write this way, and no one buys smth because it is secure.
