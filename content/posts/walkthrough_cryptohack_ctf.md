---
title: "Walkthrough: CryptoHack CTF"
date: 2021-10-03T12:41:44+02:00
tags:
  - CTFs
  - Cryptography

---

Recently I've been meaning to get into cryptography more seriously, and to be honest I've also been postponing it for a while too, so I figured it was time I wrote this article to get motivated!

I'm approaching this cryptography deep dive with https://cryptohack.org/.
Cryptohack it's website offering CTF style challenges to understand and try to break modern cryptography. I really like this gamified approach so I decided to give it a shot. 

> ***Disclaimer***
>
> You always need to be extra careful when sharing CTFs solutions online. That's the reason why I'm strictly following cryptohack's guidelines.
> As requested in the website's [FAQ](https://cryptohack.org/faq/#solutions) I'm only sharing solutions for challenges worth 10 points or less.
> These challenges are pretty basic, but I felt like it would be useful to have this kind of content published for those who are not familiar with basic cryptography or with the coding tools and technologies needed to solve the challenges. Each challenge solution will be explained but no flag will be available in this article.
>
> Cryptohack also has a functionality to share the solution once you get the flag for the challenge. Solutions to more complex challenges are to be shared exclusively there. The solutions are however only available for the solvers of the relative challenge.

Make sure to download the python notebook with the code snippets from this article [here](https://lessonsec.com/resources/cryptohack_walkthrough/cryptohack_walkthrough.ipynb).

--------------------

## Page index

- [Setup]({{< relref "walkthrough_cryptohack_ctf.md#setup" >}})
- [Introduction Challenges]({{< relref "walkthrough_cryptohack_ctf.md#introduction-challenges" >}})
  - [Finding Flags]({{< relref "walkthrough_cryptohack_ctf.md#finding-flags-2-pts" >}})
  - [Great Snakes]({{< relref "walkthrough_cryptohack_ctf.md#great-snakes-3-pts">}})
  - [Network Attacks]({{<relref "walkthrough_cryptohack_ctf.md#network-attacks-5-pts">}})
- [General Challenges]({{< relref "walkthrough_cryptohack_ctf.md#general-challenges" >}})
  - [ASCII]({{<relref "walkthrough_cryptohack_ctf.md#ascii-5-pts">}})
  - [Hex]({{<relref "walkthrough_cryptohack_ctf.md#hex-5-pts">}})
  - [Base64]({{<relref "walkthrough_cryptohack_ctf.md#base64-10-pts">}})
  - [Bytes and Big Integers]({{<relref "walkthrough_cryptohack_ctf.md#bytes-and-big-integers-10-pts">}})
  - [XOR Starter]({{<relref "walkthrough_cryptohack_ctf.md#xor-starter-10-pts">}})
- [Mathematics]({{< relref "walkthrough_cryptohack_ctf.md#mathematics" >}})
  - [Vectors]({{<relref "walkthrough_cryptohack_ctf.md#vectors-10-pts">}})
- [Symmetric Ciphers]({{<relref "walkthrough_cryptohack_ctf.md#symmetric-ciphers">}})
  - [Keyed Permutations]({{<relref "walkthrough_cryptohack_ctf.md#keyed-permutations-5-pts">}})
  - [Resisting Bruteforce]({{<relref "walkthrough_cryptohack_ctf.md#resisting-bruteforce-10-pts">}})
- [RSA]({{<relref "walkthrough_cryptohack_ctf.md#rsa">}})
  - [RSA Starter 1]({{<relref "walkthrough_cryptohack_ctf.md#rsa-starter-1-10-pts">}})
- [Diffie-Hellman]({{<relref "walkthrough_cryptohack_ctf.md#diffie-hellman">}})
  - [Diffie-Hellman Starter 1]({{<relref "walkthrough_cryptohack_ctf.md#diffie-hellman-starter-1-10-pts">}})
- [Crypto On The Web]({{<relref "walkthrough_cryptohack_ctf.md#crypto-on-the-web">}})
  - [Token Appreciation]({{<relref "walkthrough_cryptohack_ctf.md#token-appreciation-5-pts">}})
  - [JWT Sessions]({{<relref "walkthrough_cryptohack_ctf.md#jwt-sessions-10-pts">}})
- [Conclusions]({{<relref "walkthrough_cryptohack_ctf.md#conclusions">}})

----------------------

## Setup

Before starting I suggest getting the official docker image provided in the FAQs.
You simply need to pull `hyperreality/cryptohack:latest`.

To run the container simply run the provided command: `docker run -p 127.0.0.1:8888:8888 -it hyperreality/cryptohack:latest`

This will start a Jupyter Notebook server reachable at `localhost:8888`. 
If you don't want to use notebooks to solve the challenges but still want to use the container because of dependencies, you can overwrite the entrypoint of the image with the following command: `docker run -it --entrypoint /bin/bash -p 127.0.0.1:8888:8888 -v hyperreality/cryptohack:latest`

Once the docker situation is under control, we can start working on the challenges.

--------------

## Introduction Challenges

These challenges are basically tutorials to get familiar with how the challenges on this website works.
They show the flag format, how to work with the challenge scripts and how to approach the network based attacks.

#### Finding Flags (2 pts.)

Simply follow the instructions and copy-paste the flag in the text field.

#### Great Snakes (3 pts.)

For this one you need to execute the provided python script, that will print out the flag.

#### Network Attacks (5 pts.)

We need to interact with a TCP server using JSON messages.
The website suggests using python and `telnetlib` to do so. It also provides an example showing how to interact with the server of this challenge.

What we need to do to get the flag is to play around a little bit with the server and find the correct request to *buy* a *flag*.

<u>Solution</u>:

```python
# import the libraries needed for the challenge
import telnetlib
import json

HOST = "socket.cryptohack.org"
PORT = 11112

# initialize the connection
tn = telnetlib.Telnet(HOST, PORT)

# define functions to receive and send JSON payloads over TCP
def readline():
    return tn.read_until(b"\n")

def json_recv():
    line = readline()
    return json.loads(line.decode())

def json_send(hsh):
    request = json.dumps(hsh).encode()
    tn.write(request)
  
# reads the banner printed by the server
print(readline())
print(readline())
print(readline())
print(readline())

# ------ Request example ------
# Compose a request for the server
request = {"buy": "clothes"}

# Sends the request
json_send(request)

# Gets the response
response = json_recv()

print(response) # {'error': 'Sorry! All we have to sell are flags.'}

# ------ Real request ------
# mhhh flags you say?
request = {"buy": "flag"}

# Sends the request
json_send(request)

# Gets the response
response = json_recv()

print(response)
```

----------------------------------------

## General Challenges

### <u>Encoding</u>

> For these challenges it's not really necessary to write any code. While writing your own scripts can help getting familiar with tools and techniques, a deeper understanding of encodings can be obtained solving the challenges in different ways.
>
> A super-versatile and commonly used tool for this kind of task is [CyberChef](https://gchq.github.io/CyberChef/).

#### ASCII (5 pts.)

We are given a 7-bit ASCII encoded string and we need to decode it to get the flag.
The challenge hint suggests that we use the python `chr()` function to do to.

<u>Solution</u>:

```python
# ASCII values to decode
values = [99, 114, 121, 112, 116, 111, 123, 65, 83, 67, 73, 73, 95, 112, 114, 49, 110, 116, 52, 98, 108, 51, 125]

solution = ""
for v in values:
    solution += chr(v)
    
print(solution)
```

#### Hex (5 pts.)

In this challenge we are provided with an hex encoded string we need to decode.
This time the challenge hint suggests using the `bytes.fromhex()` function.

<u>Solution</u>:

```python
# HEX values to decode
values = "63727970746f7b596f755f77696c6c5f62655f776f726b696e675f776974685f6865785f737472696e67735f615f6c6f747d"

solution = bytes.fromhex(values).decode('utf-8')

print(solution)
```

#### Base64 (10 pts.)

For this challenge we are given an hex encoded string, to be decoded and then encoded in base64 to be used as flag.
In this case we will be using the `base64` python module, in particular the `base64.b64encode()`.

<u>Solution</u>:

```python
import base64

# HEX values to decode
values = "72bca9b68fc16ac7beeb8f849dca1d8a783e8acf9679bf9269f7bf"

# Decoded values
tmp = bytes.fromhex(values)
# print(tmp)
solution = base64.b64encode(tmp).decode('utf-8')

print(solution)
```

#### Bytes and Big Integers (10 pts.)

Some cryptosystems like RSA work only when applied to numbers. We need to encode our messages as numbers in order to work with these cryptosystems.
One method to do so is to represent the data as bytes and convert these in a base-16 or base-10 number.

In this challenge we are provided with a message encoded in this way and we need to get the original message out.

For this challenge the PyCryptodome library it needed, we can work with this encoding using the functions: `Crypto.Util.number.bytes_to_long()` and `Crypto.Util.number.long_to_bytes()`.

<u>Solution</u>:

```python
from Crypto.Util.number import long_to_bytes

# Message encoded as number
values = 11515195063862318899931685488813747395775516287289682636499965282714637259206269

solution = long_to_bytes(values).decode('utf-8')

print(solution)
```

---------

### <u>XOR</u>

#### XOR Starter (10 pts.)

In this challenge we need to XOR the value 13 to each character of the provided string, then we need to put the result in the cyber{flag} format.
The hint suggests that it's possible to use the `xor()` function from `pwntools` but it's just as easy to do the same in pure python.

<u>Solution</u>:

```python
# Provided string
values = "label"

solution = ""
for v in values:
    solution += chr(ord(v) ^ 13)
    
# The {{{var}}} syntax is needed to excape curly braces in python f-strings
print(f"crypto{{{solution}}}")
```

-------------

## Mathematics 

### <u>Lattices</u>

#### Vectors (10 pts.)

In this challenge we are asked to perform operations on a three dimensional vector space.
If this sounds new to you make sure to carefully read the challenge description and check the suggested materials.

<u>Solution</u>:

```python
v = (2,6,3)
w = (1,0,0)
u = (7,7,2)

def vector_minus(a, b):
   return [x - y for x, y in zip(a,b)]

def vector_dot(a,b):
    return sum([x * y for x, y in zip(a,b)])
    
def scalar_times(a, times):
    return list(map( lambda x: x * times , a))

# calculate 3*(2*v - w) ∙ 2*u
vector_dot(scalar_times(vector_minus(scalar_times(v, 2), w), 3), scalar_times(u, 2))
```

-------------------------

## Symmetric Ciphers

<u>**How AES Works**</u>

#### Keyed Permutations (5 pts.)

In this challenge we are asked to answer a question: *What is the mathematical term for a one-to-one correspondence?* 
Google is your friend for this one!                                   

#### Resisting Bruteforce (10 pts.)

This time we are asked: *What is the name for the best single-key attack against AES?*                   
Just make sure you carefully read the challenge description and you're good to go!                 

---------------------

## RSA

<u>**Starter**</u>

#### RSA Starter 1 (10 pts.)

The basis of RSA encryption is modular exponentiation. In this challenge we are asked to use such technique to create a "trapdoor function" (a function easy to calculate but hard to reverse).
This can be done using the `pow()` function that python provides.

<u>Solution</u>:

```python
# Calculate 101^17 mod 22663
pow(101, 17, 22663)
```

-----------------------------------------------

## Diffie-Hellman

<u>**Starter**</u>

#### Diffie-Hellman Starter 1 (10 pts.)

The Diffie-Hellman algorithm works with finite fields and modular exponentiation to allow to parties to exchange a shared secret.
If you're not familiar with this algorithm or with the math behind it I would suggest to check out the [Wikipedia page](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange#Cryptographic_explanation) to get started.

In this challenge we are asked to find an inverse element given the prime number and the modulo.

<u>Solution</u>:

```python
g = 209
p = 991
fc = 1

for x in range(1, p):    
    if (g * x) % p == fc:         
        print(x)        
        break
```

-----------

## Crypto On The Web

**<u>JSON web tokens</u>**

#### Token Appreciation (5 pts.)

[JWTs or JSON Web Tokens](https://datatracker.ietf.org/doc/html/rfc7519) are a standard method to safely represent claims between two parties.
This kind of token is not encrypted by default, and this is the reason why it's possible to reverse the encoding and extract the original message.

We are given the token:

`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmbGFnIjoiY3J5cHRve2p3dF9jb250ZW50c19j`
`YW5fYmVfZWFzaWx5X3ZpZXdlZH0iLCJ1c2VyIjoiQ3J5cHRvIE1jSGFjayIsImV4cCI6MjAwNT`
`AzMzQ5M30.shKSmZfgGVvd2OSB2CGezzJ3N6WAULo3w9zCl_T47K`

Now, there are a few ways to solve this challenge, the suggested one is to use Python's [PyJWT](https://pyjwt.readthedocs.io/en/stable/) library, but since it's not installed in the docker container we are using, it's easier to use an online tool like [CyberChef](https://gchq.github.io/CyberChef) or [jwt.io](https://jwt.io/).

#### JWT Sessions (10 pts.)

In this challenge we are given some information about the use of JWT tokens, now we are asked the *HTTP header used by the browser to send JWTs to the server*. Once again Google is your friend!

If you want to solve this challenge on your own, take out the developer tools in your browser, go to the network tab and start looking around for HTTP headers that could refer to the use of JWT tokens. You're *authorized* to do that!

---------------------

## Conclusions

Hope this article is useful to anyone who's meaning to get into cryptography or CTFs in general.
Writing this article allowed me to go back and review my knowledge of basic cryptography as well as exploring a bit out of my comfort zone (when it came to more complex challenges not included in the writeup).

