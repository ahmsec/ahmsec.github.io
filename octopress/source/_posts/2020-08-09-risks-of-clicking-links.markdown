---
layout: post
title: "Risks of clicking links"
date: 2020-08-09 09:21:00 -0700
comments: true
categories: 
---

We're often advised not to click untrusted links, but less often told why. This post will outline a few things that can go wrong when you simply click a link.

In brief, clicking a link can lead to exploitation of vulnerabilities in your environment. These vulnerabilities can be in your apps, computer, local network, web browser, or even in your psyche.

## Web app vulnerabilities 
A link click can exploit vulnerabilities in web applications you use. The click can send a malicious payload directly to the target web app, or it can load a malicious website, which then sends the payload. A successful attack can result in theft or modification of data from the target web app.

Examples:

- [XSS leading to compromise of Apache Foundation](https://www.acunetix.com/blog/articles/xss-to-root-apache-org/)
- [PortSwigger's top 10 techniques, published each year](https://portswigger.net/research/top-10-web-hacking-techniques)
- [HackerOne feed of unending web app vulnerabilities](https://hackerone.com/hacktivity?order_field=popular)
 
## Local service vulnerabilities
Your computer may be running local services like web servers or custom protocol handlers. These could be installed by users or by apps. A malicious website can send requests to these services and exploit vulnerabilities in them.

Examples:

- [Zoom local web server allowed websites to start your webcam](https://medium.com/bugbountywriteup/zoom-zero-day-4-million-webcams-maybe-an-rce-just-get-them-to-visit-your-website-ac75c83f4ef5)
- [Blizzard update service allowed websites to interact with it (using DNS rebinding)](https://twitter.com/taviso/status/955540415263907840)
- [Electron apps that registered protocol handlers were vulnerable to RCE](https://medium.com/hackernoon/exploiting-electron-rce-in-exodus-wallet-d9e6db13c374)

## Local network access 
Your home or office network is typically walled off from the outside world. However, when you click a link and load a malicious website, your browser executes the website's JavaScript code. This code runs inside your network (since that's where the browser is). While the code is sandboxed by your browser, it can still do things like:

- Scan the internal network for devices and ports.
- Exploit internal devices that have web interfaces, like IoT gadgets, printers, and routers. 
- Exploit internal web apps, which often have weaker security than internet-facing apps.

Examples:

- [A website can target a printer by sending requests to port 9100](http://hacking-printers.net/wiki/index.php/Cross-site_printing) and subsequently [exploit vulnerabilities](https://www.blackhat.com/docs/us-17/thursday/us-17-Mueller-Exploiting-Network-Printers.pdf) there
- [The BeEF tool has a feature that maps internal networks](https://github.com/beefproject/beef/wiki/Network-Discovery#network-map)
- Corporate networks often have internal apps that set overly permissive CORS or lack authentication. These apps may be internal tools or development versions of public apps. It's assumed that they're protected by VPN, but a malicious website can exploit them using cross-origin requests, DNS rebinding, and more. 

## Browser vulnerabilities
Your web browser may have vulnerabilities, even if it’s a modern and commonly-used browser. When you click a link and load a malicious website, the website can break out of the browser's security controls. This can allow the website to execute code on your device, install malware, or access your accounts on other websites. Browser plugins may also introduce vulnerabilities.

This is perhaps the most fearsome risk in this list. Someone can gain control over your device simply by having you visit a website. Everyone uses web browsers, so anyone can be targeted. Fortunately however, browser vulnerabilities are not easy to come by, and are quickly patched after discovery. 

Examples:

- [Chromium document describing some browser attacks: arbitrary code execution, UXSS, and Spectre-like side channel attacks](https://www.chromium.org/Home/chromium-security/site-isolation#TOC-Motivation) (in the Motivation section)
- [Coinbase article on how they were targeted by Firefox exploits](https://blog.coinbase.com/responding-to-firefox-0-days-in-the-wild-d9c85a57f15b)
- [Evernote plugin for Chrome allowed malicious websites to access other websites](https://guard.io/blog/evernote-universal-xss-vulnerability)

## Privacy
Clicking a link and visiting a website may expose your personal information. This can include your IP address, geolocation, operating system, language, browser information, and more. This information can be unique to you, and may be correlated with activity on other websites.

Example: 

- [EFF website that demonstrates what private information can be leaked](https://panopticlick.eff.org/) (Click "Show full results for fingerprinting" after the test)

## Phishing 
Clicking a link can take you to a webpage that tricks you (or "phishes" you) into entering your credentials, downloading & running malware, or compromising you in some other way. Good attacks can appear convincing and legitimate.

While phishing could require more than just clicking a link, it is perhaps the most common risk of clicking links, and is frequently the first step in a broader attack. 

Examples:

- [Phishing campaign targeting O365 OAuth](https://www.bleepingcomputer.com/news/security/phishing-attack-hijacks-office-365-accounts-using-oauth-apps/)
- [Citizen Lab report on a hacking operation that largely used phishing](https://citizenlab.ca/2020/06/dark-basin-uncovering-a-massive-hack-for-hire-operation/)
- [How Microsoft Office files can contain malware](https://docs.microsoft.com/en-us/windows/security/threat-protection/intelligence/macro-malware)

## Mitigations
Given all that can go wrong, what can you do about it?

It comes down to basic security hygiene. Use U2F/2FA, use a password manager, keep all software up-to-date, avoid installing unnecessary software, limit app permissions to what is necessary, log out of accounts you're not using, and put untrusted "smart" devices on separate network segments. 

To the extent practical, exercise caution with links, especially if unsolicited. Check if the domain is one you trust or expect. Be careful of what you enter, approve, download, or run.

Finally, security practitioners need to ensure that their systems and environments are resilient to users clicking on malicious links. It is unreasonable to expect users never to click on malicious links. Typical internet usage involves clicking many links, and good phishing scenarios are difficult to distinguish from legitimate scenarios.
