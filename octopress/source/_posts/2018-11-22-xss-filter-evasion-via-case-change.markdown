---
layout: post
title: "XSS Filter Evasion via Case Change"
date: 2018-11-22 13:00:00 -0800
comments: true
categories: 
---
[This](https://twitter.com/x00x90_/status/1060397253024727040) tweet describes an interesting behavior: certain non-ASCII characters map to ASCII characters when converted to upper- or lower-case. Specifically: 

    ı (\u0131) to upper-case --> I
    ſ (\u017f) to upper-case --> S
    İ (\u0130) to lower-case --> i
    K (\u212a) to lower-case --> k

This can help bypass XSS filters or blacklists. For example, the filter in the app below can be bypassed by `?name=<ſcript src="/alert1.js"></script>`. 

``` python vulnerable-app.py
from flask import Flask, request, Response
app = Flask(__name__)

@app.route("/")
def main():
    name = request.args.get('name') or 'guest'
    if '<script' in name.lower():
        return 'XSS DENIED!'
    return '<html>Welcome, ' + name.upper() + '!</html>'

@app.route("/ALERT1.JS") #normally hosted on attacker site
def alert1():
    return 'alert(1)'
```

As usual, the best practice for XSS prevention is [character encoding](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet). 

Here is a quick script to enumerate characters affected by this behavior. Interestingly, it appears that Python 2 and 3 treat `İ (\u0130)` differently. 

``` python casing-xss.py
import struct
  
interestingChars = [chr(i) for i in range(65,91)] + [chr(i) for i in range(97,123)]

for i in range(0x10FFFF+1):
    char = struct.pack('i', i).decode('utf-32', 'surrogatepass')

    charLower = char.lower()
    if charLower in interestingChars and char not in interestingChars:
        print('{} ({}) to lower-case --> {}'.format(char.encode('utf8'), repr(char), charLower))

    charUpper = char.upper()
    if charUpper in interestingChars and char not in interestingChars:
        print('{} ({}) to upper-case --> {}'.format(char.encode('utf8'), repr(char), charUpper))
```        