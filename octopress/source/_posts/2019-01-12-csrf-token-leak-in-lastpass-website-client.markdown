---
layout: post
title: "CSRF Token Leak in LastPass Website Client"
date: 2019-01-12 17:00:00 -0800
comments: true
categories: 
---

Here is a bug I reported to LastPass, copied below with some edits. They shipped a fix within 4 business days and paid out a $1k bounty. 

One takeaway is the need for defense-in-depth. While our development standards may prohibit sensitive tokens in URLs, it can still happen due to human error. We can mitigate this by adding a strict [Referrer Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy) like `<meta name="referrer" content="no-referrer">`. 

A second takeaway is the importance of secure defaults. In this scenario the top-level frame actually had an `origin` Referrer Policy. However, it did not prevent the vulnerability because the browser didn't [inherit](https://www.w3.org/TR/referrer-policy/#referrer-policy-delivery-nested) the policy to child iframes. A more secure default could perhaps be for browsers to inherit Referrer Policy to child iframes if they're on the same origin and don't specify their own policy. 

## LastPass Bug Report

### Summary
The website client (https://lastpass.com) leaks CSRF tokens to saved websites. This happens because the iframe containing "Launch" buttons has the CSRF token embedded in the `src` attribute. 

``` html
<iframe id="newvault" style="border: 0px; width: 100%; height: 100%;" src="newvault/vault.php?noscript=1&amp;fromindex=1&amp;ac=1&amp;lpnorefresh=1&amp;fromwebsite=1&amp;newvault=1&amp;nk=1&amp;xmlerr=1&amp;token=[redacted]"></iframe>
```

This vulnerability affects Firefox and potentially other browsers, but does not appear to affect Chrome (not sure why, could be browser referrer policy inheritance or different app codebases). 

### Impact
A malicious webpage can make various state-changing requests to LastPass, and LastPass will honor them. 

Affected endpoints include changing password hint and changing master password (only resulting in account lockout, not credential leak). 

There are likely to be more affected endpoints. 

### Reproduction steps
1. Log into the Firefox website client on https://lastpass.com.
2. Add a new site and set the URL to https://www.example.com. 
3. Click 'Launch' on the new site. 
4. On the launched page, open your browser dev console and type `document.referrer`. 
5. Notice the LastPass CSRF token included in the referrer. This is readable by target sites.

If anything doesn't work, be sure you set the target to an HTTPS site, used Firefox, and used the website client and not the extension. 

### Proof of concept
1. Use Firefox to log into https://lastpass.com.
2. Add a new site and set the URL to [https://poc.ahmsec.io/3302aa361e2a44ca9bb82d66bfedd243/lastpass-csrf.html](https://poc.ahmsec.io/3302aa361e2a44ca9bb82d66bfedd243/lastpass-csrf.html).
3. Click 'Launch' on the newly added site. 
4. Click 'Change my password hint'. *A real exploit would do this step silently without prompt.* 
5. Check your password hint and notice that it changed to "i-got-hacked".

### Potential attack scenarios
* Victim generates & saves creds for some shady one-off online store. Online store is either compromised or malicious. When victim launches site from LastPass, the site can carry out CSRF attacks against victim's LastPass. 
* Malicious user shares site with victim on LastPass. Victim launches site to check it out, and is attacked via CSRF. 

### Suggested mitigations 
1. Send CSRF tokens in request body or header instead of URL parameter. 
    * URL parameters have higher risk of leakage via referrers and logs. 
2. Add a `<meta name="referrer" content="no-referrer">` referrer policy to https://lastpass.com/newvault/vault.php. 
    * This policy is included on the parent frame, but not in the vault.php nested iframe.
3. On highly sensitive endpoints, *require* password hash in addition to CSRF token.
    * E.g. while password changes do require entering old password, the final POST request can be made without it. (See sample request.)
4. Restrict CSRF tokens to sessions. Do not let them be reused across sessions.  