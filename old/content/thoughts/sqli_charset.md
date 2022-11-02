---
title: "Bypass PHP addslashes()"
date: 2021-02-12T12:03:55+01:00
---

So I was working on a CTF and had to try to perform an SQLi through a PHP page on an `addslashes()` sanitized text field.
Turns out the flaw was in a business logic and I was looking in the wrong spot, but I found this interesting article on bypassing this sanitization mechanism abusing GBK character encoding: https://shiflett.org/blog/2006/addslashes-versus-mysql-real-escape-string
