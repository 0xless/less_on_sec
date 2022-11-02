---
title: "Chunked"
date: 2021-03-01T18:07:12+01:00
---

Was working on something and got curious about the `chunked transfer encoding`, in particular about the fact that HTTP headers could be defined after the body of the request. Dug a bit deeper and found out it's actually a known attack vector! Check this for more: https://swende.se/blog/HTTPChunked.html