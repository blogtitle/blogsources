---
title: "Rob'n Go security pearls: Cross Site Request Forgery (CSRF)"
date: 2019-09-06T16:20:23+02:00
categories: ["Web security"]
tags: ["web","security","golang"]
authors: ["Rob"]
draft: true
---
> This post is part of my security training for developers. You can find all other posts [here](https://blogtitle.github.io/categories/web-security/).

# Preface: same origin policy
In the previous two posts I explained [some fundamentals](https://blogtitle.github.io/robn-go-security-pearls-fundamentals/) and [what cookies are and how they work](TODO link). In this post I'm going to describe one of the many scenarios in which browsers do not completely protect one web application from another one.


