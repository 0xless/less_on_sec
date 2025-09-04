---
title: "Dev Chronicles: Creating an HTTP attack client"
date: 2021-04-01T23:23:00+02:00
toc: false
tags:
 - Dev Chronicles
 - HTTP
image:
 - /images/intrudo/logo.jpg
---

Intrudo is a tool for automating customized attacks against web applications loosely shaped after [burp intruder](https://portswigger.net/burp/documentation/desktop/tools/intruder).  

| ![intrudo logo](/images/intrudo/logo.png#center) |
| :----------------------------------------------: |
|                   Intrudo logo                   |

Check it out: [https://github.com/0xless/Intrudo](https://github.com/0xless/Intrudo)

### A little of background

While working on CTFs I would usually write a bunch of lines of python to automate attacks, it worked fine, but I felt like I needed to improve my workflow. So I decided to learn new approaches and strengthen my use of commercial offensive tools.  
I have used burp suite in the past, but I have not taken the time to explore it thoroughly, so I figured the best way to do that would be the labs that [PortSwigger created for this reason](https://portswigger.net/web-security).

The idea of developing this tool came to me when working on SQL injection challenges. The labs and the learning material were great, but the throttling on the requests in burp intruder (community version only) was making it difficult to complete these challenges.  Due to the throttling, completing a challenge would take several hours since for every request sent, you have to wait more and more. 

Every challenge was focused on retrieving an administrative password, in some of these challenges a request needed to be sent for every possible letter and digit for each character of the password.  Now these challenges could be solved in two ways. You could either send requests to discover one character at time or try to automate it using the "cluster bomb" attack. In the first scenario, you'd have to adjust the parameters and manually launch the attack every 30 minutes or so. In the second scenario, you could automate the whole thing, if the challenge password didn't change mid-attack because the client took so long to send the requests.

One would assume these challenges were developed *specifically* to make it harder to solve for those who didn't have a paid version of the software. I'm not implying they have a malicious intent, but they could have made it easier for *everybody* to solve the challenges and learn from the awesome materials they share.

After my experience with PortSwigger's labs, I decided to develop Intrudo; a fast, throttling free alternative to burp intruder.   

### The initial approach

My first idea was to use multithreading so I started surfing the web for ideas and found [aiohttp](https://docs.aiohttp.org/en/stable/), and asynchronous HTTP Client/Server based on [asyncio](https://docs.aiohttp.org/en/stable/glossary.html#term-asyncio). It wasn't multithreading but it seemed perfect for this project.

Unfortunately, I was not lucky with the choice of developing around `aiohttp` as the client was strict on what requests can be sent and how they are sent. 

I learned this the hard way, of course. I spent a few days developing with `aiohttp` and all I could achieve was an incredibly fast fuzzer that would only work on URL parameters. Big let down.

At this point I could try two things:

- see if I could find some libraries I could fork
- implement my own client consisting of bare bones sockets and some good ol' `asyncio` magic

### The wrong way

So I decided to try and modify some libraries first. It turns out every library that implements an HTTP client wants to ensure it is compliant and do so by thoroughly reviewing the data an user would want to send.
Usually these libraries make you specify method, URL, headers and body of the request separately. 

While this approach with HTTP complaint requests works well for standard clients and preventing errors, it is not flexible enough for this type of project.  This makes it incredibly hard to implement complex attack logic, work with headers and cookies and send non-HTTP compliant malformed data. 

After looking around, studying the HTTP protocol specifications and coding for a few hours, I was able to read some requests (non-chunked transfer encoding ones) and manage content compression!

Well, to be completely honest, I borrowed and modified some code from [urllib3](https://github.com/urllib3/urllib3/blob/main/src/urllib3/) to handle the decompression. 

I did this for two reasons: 

- the code is clean, simple and well tested
- someone has already considered aspects of the HTTP standard I didn't even know existed

Since `urllib3` worked so well I decided to try and use as much code as I could.  
One tweak here and one tweak there and I got myself into completely modifying `urllib3`... Bad idea!  
`urllib3` offers great client functionalities, but it's too high level and wouldn't let me work with `asyncio` streams to read data.

`urllib3` requests are made by `http.client`, a standard python library. So guess, what I tried next? You guessed correctly, I attempted to hack `http.client` to work with `asyncio`.
I knew from the beginning that it wasn't a smart idea, and it turns out I was right: modifying the `http` library to suit my needs is an hard task. `http.client` would only read file-like objects and there was no way I could make it to use `asyncio` streams (without rewriting a huge portion of the code, of course).

If you can't beat 'em, join 'em, right? So I decided to "transform" `asyncio` streams into files with the hope `urllib3` will come to like them.  At this point you may think: "*less, if you read the whole content of a socket, the client would not be asynchronous anymore!*", and my answer to that would be ~~"*mind your own business*"~~ "*we can manage sockets in some kind of spooled way!*". But of course that didn't pan out.

After a few hours of unsuccessful attempts, I decided to take a step back and code the whole client from scratch.

### The right way?

I quickly realized it wasn't as easy as I expected it to be, but I did find some [well written documents](https://www.jmarshall.com/easy/http/) that helped me better understand what I was I was looking for. So, the custom HTTP client adventure begins!

I was able to put together some code I had already written and the decompression module from `urllib3`. That resulted in an HTTP 1.0 semi-compliant client (it would have been nice if HTTP 1.0 was used anytime in history, but sadly it has not).
I want to make it HTTP 1.1 compliant to the extent that I need the client to be compliant, so the only thing missing would be to manage
`chunked transfer encoding`.

After avoiding this for a whole week, I realized it was easier than I expected. If I hadn't wasted so much time trying to poke at the libraries, I probably would have had a nice piece of client already. 
But you can't cry over spilled milk, so let's get back to work! After 1 hour of coding and 1 hour of debugging, I was able to achieve HTTP 1.1 compliance!

And just like that "cliente" was born: a fully asynchronous, unrestricted, HTTP attack client. Simple yet effective.

### The right way!

Everything seemed to work smoothly after the custom client was complete.
I implemented the payload position logics just like in burp intruder (pitchfork, sniper, cluster bomb and battering ram), it was not easy to work with two delimiters: I had to play around a little bit with `regex` and had to work with payload position offsets given by the substitution of a text with a payload. To this day I'm sure there must be a better solution.

After that I implemented a few simple callback mechanisms to manage, analyze and store incoming responses which match user specified criteria.

It's only a draft, but the very first version of Intrudo was complete!

Here's a little demo of the speed Intrudo can offer at the moment:

![intrudo demo](/images/intrudo/demo.gif#center)

### What I learned

Of course the main takeaway is a deeper understanding of the HTTP protocol, especially when it comes to encodings and compression methods, since I  had to work with these directly.

Working with raw HTTP requests was also the main challenge, ensuring that an user specified request is acceptable by the server was an hard task. Since there are no debug messages this process was based on a "try and fail" method and I had to read multiple rfc documents on the standard but I could complete this task, at least for the majority of servers I've experimented with.  

Another interesting thing I got from this project is a grasp of asynchronous programming. I had studied it in theory but learning a framework to actually write asynchronous programs is different thing.  `asyncio` is a powerful tool I will be using in other projects.

Finally I had to mess around with `regex` and learned about greedy `regex`. Handling the placeholder delimiters in the requests wasn't straightforward and for the sake of performance (to avoid recomputing placeholder positions for each payload inserted) I had to develop a module that would insert the payload in the desired position considering the offsets from previously inserted payloads. 

### To be continued

There are a few things that need to be done:

- The callbacks are not as complete as burp intruder ones.
- There is no payload generator (but that's because Intrudo is to be used as a framework, not with a GUI).
- Some servers will always respond with "bad request" (can't figure out why).
- Some serious testing is yet to be done.
- The timeout on requests is to be calculated with an adaptive function to optimize the throughput. 