---
layout: post
title: "Custom Headers as CSRF Defense"
date: 2016-07-03 16:18:01 -0700
comments: true
categories: 
---

CSRF is a prevalent and well-known vulnerability that affects web applications. The common way to protect against CSRF is to require anti-CSRF tokens on state-modifying requests. For defense in depth, you can add an extra layer of security by additionally requiring custom headers. This can mitigate scenarios where anti-CSRF tokens are somehow leaked (something I have seen happen). 

### Simple vs Preflighted Requests

The Mozilla [article](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) on CORS does a nice job of explaining the difference between "simple" and "preflighted" requests. 

 * **Simple requests** are sent by traditional mechanisms on the web, such as GETs made by `<img>` tags or POSTs made by HTML forms. The browser will always send them cross-origin, and will always pass along session cookies. This behavior is what makes CSRF possible. 

 * **Preflighted requests** are requests that either contain custom headers or use methods other than GET, POST, or HEAD. Browsers will refuse to send these requests cross-origin unless the server explicitly allows it. To check this permission, browsers will send a "pre-flight" OPTIONS request before sending the actual request, and the server has to reply with the appropriate CORS headers. If the server doesn't, the actual request is never sent. 

 Note the difference: simple requests are always sent cross-origin, even though Same-Origin Policy blocks the responses. When preflighted, the request isn't even sent in the first place. 

### Preflighted Requests and CSRF

If a resource requires a non-standard header or method, it won't be sent cross-origin (unless the victim domain for some reason whitelists the attacker domain). Since the request is never sent, CSRF attacks are blocked.

### Bypassing Custom Header Restrictions

While requiring custom headers is a useful layer of CSRF defense, it shouldn't be the only one. Up until March 2015, Adobe Flash had a vulnerability that allowed you to send custom headers cross-origin. Here are two well-written posts describing the technique: 

   * [http://lists.webappsec.org/pipermail/websecurity_lists.webappsec.org/2011-February/007533.html](http://lists.webappsec.org/pipermail/websecurity_lists.webappsec.org/2011-February/007533.html)
   * [http://www.appsecweekly.com/flash-same-origin-policy-bypass-with-307/](http://www.appsecweekly.com/flash-same-origin-policy-bypass-with-307/)

### Conclusion
Consider requiring custom headers on your sensitive state-modifying requests. For example, you can require an X-Requested-With header on AJAX calls and refuse to honor those that don't include it.   
