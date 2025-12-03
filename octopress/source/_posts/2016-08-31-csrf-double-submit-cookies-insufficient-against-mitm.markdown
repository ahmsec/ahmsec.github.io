---
layout: post
title: "CSRF: Double Submit Cookies Insufficient Against MitM"
date: 2016-08-31 08:38:05 -0700
comments: true
categories: 
---

Suppose you’re at the latest hip coffee shop in town, enjoying their comfy chairs and high-speed wifi. You’re doing some confidential work using a relatively secure web application. The application always uses TLS, redirects HTTP requests back to HTTPS, and deploys [Double Submit Cookies](https://www.owasp.org/index.php/Cross-Site\_Request\_Forgery\_%28CSRF%29\_Prevention\_Cheat\_Sheet#Double\_Submit\_Cookie) (with the ‘secure’ cookie flag) to protect from CSRF. Now suppose that the coffee shop staff becomes part of your threat model. Could they still launch CSRF attacks against you? After all, they can’t read TLS-encrypted traffic, so they can’t possibly steal CSRF tokens.

It turns out that actually yes, given the scenario above, a man-in-the-middle (with your pick of malicious staff, rogue access point, or ARP poisoning) can defeat the Double Submit defense. And it’s a simple attack, not one that requires you to break TLS or WPA2. Here’s how it works: 

1. Victim conducts secret business on https://secret.example.com (MitM attacker can’t read this traffic.)
2. Victim visits a non-SSL website like http://stackoverflow.com or http://www.cnn.com (MitM can read and modify this traffic.)
3. MitM injects `<img src=”http://secret.example.com/non-existent.png”/>` into the response from (2). Note that the img tag’s protocol is HTTP and not HTTPS.
4. Victim’s browser receives the image tag, and sends a secondary non-SSL request to http://secret.example.com/non-existent.png. (CSRF cookie not sent because of secure flag.)
5. MitM intercepts the response from http://secret.example.com, which could be a 301 redirect [1] or a 404 error, and injects a Set-Cookie header that overwrites the victim’s CSRF cookie with a known value. Details on how later.
6. Since the attacker knows the CSRF cookie value, he or she can now trigger a request that includes the CSRF token in the body. 

All of this can be automated for your exploit. A non-intuitive step in this attack scenario is perhaps #5. How could a non-SSL response possibly overwrite cookies that have the ‘secure’ flag and that were set under SSL? That unfortunately is allowed by design and is how the web works today. Here’s an article that describes the issue: http://scarybeastsecurity.blogspot.com/2008/11/cookie-forcing.html . 

One way to protect against this scenario is to use HSTS. However, you have to make sure that every single subdomain has HSTS, because any child subdomain can be used to set cookies for the top-level parent domain, and those cookies will subsequently be sent with requests to every subdomain. You also have make sure you don’t have users on browsers without [HSTS support](http://caniuse.com/#feat=stricttransportsecurity), such as IE < v11. 

A few takeaways:    

1. Double Submit Cookies are a common way to protect against CSRF. They’re recommended by OWASP and used by several websites and frameworks. However, if you need to protect against MitM, you should consider a different defense.  
2. Cookies should be treated as untrusted and attacker-controlled input. Be careful of mistakes like allowing user-generated session cookies and not sanitizing cookie-sourced values for XSS.

For further discussion on these topics, please see the following articles.   
&nbsp;&nbsp;&nbsp;&nbsp; - http://scarybeastsecurity.blogspot.com/2008/11/cookie-forcing.html  
&nbsp;&nbsp;&nbsp;&nbsp; - https://media.blackhat.com/eu-13/briefings/Lundeen/bh-eu-13-deputies-still-confused-lundeen-wp.pdf

--------
[1] For a workaround to browsers’ aggressive caching of 301 redirects, see http://security.stackexchange.com/a/117138. 

