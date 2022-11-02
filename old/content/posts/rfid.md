---
title: "Cloning RFID tags for fun and profit"
date: 2021-04-20T16:27:00+02:00
toc: false
images:
  - /images/rfid/t55xx.png
  - /images/rfid/bricked.jpg
  - /images/rfid/cloned.jpg
tags:
  - RFID
  - Radio
  - Physical security
image:
  - /images/rfid/main.jpg
---

RFID tags are a technology commonly used but not limited to industrial purposes. These systems are in fact used every day as public transport passes, as security token in access control systems or as a digital "bar code" in shops.
Given the diffusion these tags have, it's important to understand how they work and the security implication of their use.

>Radio-frequency identification (RFID) uses electromagnetic fields to automatically identify and track tags attached to objects.  
>An RFID system consists of a tiny radio transponder, a radio receiver and transmitter. When triggered by an electromagnetic interrogation pulse from a nearby RFID reader device,  the tag transmits digital data, back to the reader. - [Wikipedia](https://en.wikipedia.org/wiki/Radio-frequency_identification)

![rfid tools](/images/rfid/main.jpg)

### What's RFID?
RFID is a set of standards and technologies because it includes multiple frequency ranges such as:
- LF: 120–150 kHz
- HF: 13.56 MHz
- UHF: 433 MHz
- UHF: 865–868 MHz (Europe) 902–928 MHz (North America) 
- microwave: 2450–5800 MHz
- microwave: 3.1–10 GHz 

In practice, only tags operating in the LF range are commonly called RFID tags while tags operating in the HF range are called NFC tags.  
The rest are radio technologies not limited to the near field use (i.e. bluetooth).

There's a big difference between RFID and NFC tags and it hides in the specifications. They operate on different frequencies, use different protocols, offer different features and have different uses. Some RFID devices can be compatible with NFC readers, but that doesn't mean that they strictly follow the NFC specs. Further considerations on the difference between these technologies are out of the scope of this article, but it's important not to confuse the two.

LF tags operate in the 120–150 kHz range, but the most commonly used frequencies are 125kHz for access control tags and 134kHz for uses like pet chips. Other frequencies can be used but the vast majority of tags use either 125kHz or 134kHz. Such low frequencies can limit the data transmission speed. 

RFID LF tags can be passive. This means that the tag is powered and interrogated by the reader. Or active, meaning that the tag is powered with a battery and that it continuously broadcasts data.
While active tags are used, they often have specific purposes. This article will focus on RFID LF passive tags since it's the most common variant found in everyday life.

RFID LF tags have poor transmission speed but are incredibly cheap to produce. Due to this, they are so widely employed where speed is not fundamental.
The reading distance of LF tags is usually better compared to HF reading distance. In fact, LF is referred as a vicinity technology, while HF is generally called a proximity technology. 

Industrial uses aside, one of the main uses of these tags is as access control tokens. It's common to see these tags in form of badges or key fobs, and these can be used to access homes, offices or critical infrastructures.

There are numerous models of RFID LF tags each with it's specific features and peculiarities, but the use as a security token is not ideal for one main reason: RFID LF tags can be an insecure option. (note that are exceptions: some tags can employ a password or crypto mode, one of the very few examples are [Hitag2](https://www.nxp.com/products/rfid-nfc/hitag-lf:MC_42027) tags and these security measures [can still be circumvented](https://www.cs.bham.ac.uk/~tpc/isecsem/talks/EZ.pdf)!)

In this article, we will see how it's possible to read, write and clone these tags and learn about possible implications due to misuse of this technology. 

### How to work with RFID tags
To work with RFID tags, specialized hardware is necessary.

Different devices exist:

- Chinese cloners  
  Simple devices that read from one tag and write on another.
  Generally, these are hand-held, feature read and write buttons, and have a couple of status LEDs.
  Some are more advanced than others and can feature a little screen.
  Compatibility for frequencies and standards may vary, but these devices are usually good enough for simpler tasks.
  
- RFID Chameleon  
  Developed to be used in RFID security assessments.
  Doesn't offer the most advanced features, but is designed to be used in the field, is battery powered and supports tag simulation and manipulation.
  It is also programmable and you can use it with an app.
  
  It's probably the best solution for a standalone use.  
  More here: https://kasper-oswald.de/gb/chameleonmini/
  
- Proxmark3  
  As the website puts it: Proxmark is an RFID swiss-army tool.
  It represents the state of the art when it comes to RFID research.
  It allows interacting with the tags both high and low level.
  Different versions have different features, including bluetooth support, battery packs,
  swappable antennas and so on.
  
  It supports a standalone use, but it's more powerful when connected to a PC. 
  It's to be intended  as a research tool.  
  More here: https://www.proxmark.com/
  
- Generic boards  
  Other boards exist. These are usually sold as an Arduino add on, but dongles featuring
  the same integrated circuits are available.
  These are not widely used outside the makers world, but many are compatible with
  [`libnfc`](http://www.nfc-tools.org/index.php/Libnfc) and can be useful to perform simple to advanced operations.

The only device that I have available is a Proxmark3 easy (cheap Chinese version) so this
article will focus on its use. The underlying concepts about RFID tags should translate to other devices.

### Tags that emulate other tags
When it comes to working with RFID LF tags, there's one main player: the [t55xx](http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-9187-RFID-ATA5577C_Datasheet.pdf) tag

They are a family of tags developed to emulate a wide range of regular tags.
This means that it's possible to clone most of the RFID tags around using one of these
without having to carry writable cards for each and every tag model.

This tag features 8 x 32 bit blocks in page 0 and 4 x 32 blocks in page 1.
Page 1 blocks are meant to be used for configuration purposes along with block 0 and 7 in page 0.
Blocks 1 to 6 in page 0 are dedicated to user data.

![t55xx](/images/rfid/t55xx.png#center)

Of course, it's possible to work directly on the configuration blocks, and this is what allows the emulation
of other type of tags, but doing so carelessly can easily lead to a brick! 

![bricked tag](/images/rfid/bricked.jpg#center)

Original Atmel T5577 tags have a test mode and that can be helpful to recover soft-bricked cards.

### Cloning tags
Using the proxmark3 CLI, reading and writing devices is pretty straightforward.
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

It's an HID Prox tag, let's try and read its ID:
```
[+] [H10301] - HID H10301 26-bit;  FC: 118  CN: 1603    parity: valid
[=] raw: 000000000000002006ec0c86
```

And just like that we got the devices ID. That ID is the authentication token!
If we can write this token on a t55xx tag, we can emulate the card and possibly gain access
to a restricted perimeter.

Let's see how it's done.  
First we need to get a t55xx tag and position it on the reader. When empty this device
can't be found using `lf search`, so we can make sure it's a t55xx tag using the detect command:

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
The ` lf t55 detect ` command is also necessary before using this tag because it detects the configuration in use and helps avoiding problems running other `lf t55` commands.

Now we can see the content of the device:

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

Note that it's possible to operate on single blocks too, but for the purpose of this article, it's easier to dump the whole memory instead.

We see that the card doesn't contain user data. If it did, wiping the card with the `lf t55xx wipe` command would be suggested.
We can now try and emulate tags on it!
To do that we only need to have the ID of the tags we want to emulate. We already saw how that's done.

We can now go ahead and clone the tags, let's try the em410x first:

```
[usb] pm3 --> lf em 410x clone --id 1122334455
[+] Preparing to clone EM4102 to T55x7 tag with ID 1122334455 (RF/64)
[#] Clock rate: 64
[#] Tag T55x7 written with 0xff8c65298c94a940
```

Once the command is issued, we can read the device and verify that it emulates an em410x tag:
```
[usb] pm3 --> lf em 410x reader
[+] EM 410x ID 1122334455
```

Of course it's still a t55xx tag (and the `detect` command will tell you that) but it behaves exactly like and em410x.

Now we can try with the HID Prox.
```
[usb] pm3 --> lf hid clone --r 000000000000002006ec0c86
[=] Preparing to clone HID tag using raw 000000000000002006ec0c86
[=] Done
```

And of course reading it reveals that we successfully cloned the tag:
```
usb] pm3 --> lf hid reader
[+] [H10301] - HID H10301 26-bit;  FC: 118  CN: 1603    parity: valid
[=] raw: 000000000000002006ec0c86
```

We now have two key fob tags copied on t55xx cards!

![](/images/rfid/cloned.jpg)

In this demo only HID Prox and em410x tags are examined, but it's possible to clone and work with many more of these tags.

Since it's possible to emulate cards knowing the ID, we can clone some RFID LF cards "by sight" simply reading the ID printed to the device body. 
This completely removes the limit of having to read the card with a specialized tool. 
Some of these printed ID are "encoded" (or shifted by some value). This allows organizations to "decode" it, but prevents attackers from obtaining the ID by sight.

At this point you might be wondering if it's THAT easy to clone a tag in the real world, the answer is no. That's for a simple reason, the tag is passive and if we use the standard antennas provided with whichever device, the reading range is limited to a few centimeters.

Luckily, it's possible to weaponize a bigger antenna! If we use a bigger and more powerful antenna, it's possible to clone a LF tag from a usable distance. Of course the antenna will need to be powered by a big battery pack and carried in some kind of backpack or messenger bag, but that's the price to pay.

More here: https://www.youtube.com/watch?v=wYmVtNQPlF4  
(NOTE: many other implementations exist!)

### Defeating password protection
When examining the t55xx tag, you may have noticed a parameter "Password set". That's because
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

But it does if the correct password is specified:

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

While this type of attack works, it's a really slow and instable method. This means it's hard to try all the possible passwords before the connection drops. It may seem that the password protection is effective and that's true if you don't have the right tools. On the other hand, with quality reading devices and unlimited access to the target tag, success is granted.

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

As you can see, it took ~20 minutes to crack a 4 hex digit password. It may seem to be a reasonable time but we must consider that the password can be double the length and that having access to the card for long periods of time is not always an option.

### Attacks on RFID readers
Finally, we can talk about attacks on the readers. There are two main attacks:
- Tag simulation
- Data exfiltration

We saw that we can simulate a tag using the proxmark, but what if the ID we have read and simulated doesn't have enough permission to grant us access somewhere?
In this scenario, it's possible to use the read value as an upper limit to the ID space to research, and simulate every ID lower than that value hoping to find an ID with higher privileges. This attack is based on the assumption that lower IDs may have higher privileges because such privileges are associated with users registered earlier into the system, hence the lower ID value.
As for the bruteforce attack, this process may take a while, so it's not always a usable technique.

A sneakier way to get valid IDs is to install a device such as the [ESPKey](https://redteamtools.com/espkey) inside the reader.
This approach allows the interception of data directly from the wires and can harvest valid IDs.
Of course, this requires a physical access to the reader.

Using the proxmark is also possible to sniff data from a tag to a reader.  
Given the antenna range, this is not a widely used technique and personally I haven't tried it yet.

### Conclusions

This article shows how, with the right hardware, it is possible to clone RFID tags with relatively low skills.  

What's scary is how easy it is to "steal" valuable credentials. In an access control context, a stolen ID constitutes a danger for a business because of the implications of a possible unauthorized entry into a critical infrastructure.

Possible mitigations include using a stronger authentication mechanism and trainings on RFID security and how to keep access tokens safe. Simple precautions like using an RFID shield (test those! some doesn't work!) and avoiding to keep a tag in sight can often be a huge improvement in access token security. 

A secure reader is also crucial. Some readers offer tamper detection mechanisms and actively try to detect and disable rewritable tags to avoid unauthorized entry.

RFID credentials cloning can be one of the strongest tools in the arsenal of a physical penetration tester.

--------------------------------------------------

This article was reviewed by an external source, big thanks to them!



