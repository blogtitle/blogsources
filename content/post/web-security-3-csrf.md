---
title: "Rob'n Go security pearls: Cross Site Request Forgery (CSRF)"
date: 2019-09-12T16:20:23+02:00
categories: ["Web security"]
tags: ["web","security","golang"]
authors: ["Rob"]
draft: true
---
> This post is part of my developer-friendly security series. You can find all other posts [here](https://blogtitle.github.io/categories/web-security/).

Cross-Site Request Forgery is usually shortened to CSRF(pronounced see-surf) or XSRF. CSRF has a generally more limited impact than [XSS](TODO link) but it shouldn't be underestimated: this vulnerability allows the attacker to act in stead of the victim and perform some state-changing actions without their knowledge. These actions could be account deletion, money transfer or stuff orders. A victim could be visiting an online gallery of cat pictures and also unknowingly be ordering pizza for the site owner.

In this post we'll talk about some browsers protections against these attacks and how to properly secure a web service when those protections fail.

# Same-Origin Policy
Browsers are user agents for the web. This means that they act in behalf of the user to interact with web services. As **the** user agent a browser has to make sure the user is not just accessing content but that they are also as safe as possible when exploring the wilderness of the internet.

One of the many features we will talk about in this area is the **Same-Origin Policy**. This concept is convoluted and has many different manifestations so we will come back to this in other posts. The gist of it is that the browser will restrict pages hosted on different "servers" from doing things to each other. If you are curious to know exactly what an origin and a site are, you can take a look [here](TODO link).

Here is a brief introduction on what an origin and a site are:

Since the web organically grew from being just a heap of documents with links to the ecosystem that it is today, proper security was not part of the original design. Some features were added just because they made developers lives easier, without consideration on the consequences they carried with them (this still happens today even if in a much more limited way).

At one point in time servers started having logic and support things like authentication and restricted resources, but already existing web applications were relying on the possibility to submit forms cross-site. Browser vendors decided to **maintain backward compatibility and allow some cross-site requests to be performed**.

# Cross-Site request forgery
If a page has this html code:
```html
<form action="https://pizza-delivery.science/order_pizza" method="POST">
  <input type="text" name="quantity" value="10"><br>
  <input type="submit" value="Submit">
</form>
```

When the user hits the "submit" button the browser will send a request to `https://pizza-delivery.science/order_pizza` **regardless the origin of the current page**. For the server this request will be **indistinguishable** from one sent by a page hosted on `pizza-delivery.science`. Moreover forms can be hidden in a frame and automatically submitted on visit, so the user wouldn't even notice they ordered pizza with pineapple until it's too late and the attacker is done consuming their crime against humanity.

Some might argue that the browser will send a `Referer` header with that information but it is not reliable and sometimes it is not even sent, so it is not a good way to deal with this issue. 

To prevent a site hosted on `hungry-attacker.com` from ordering pizza on `pizza-delivery.science` we need to get creative.

