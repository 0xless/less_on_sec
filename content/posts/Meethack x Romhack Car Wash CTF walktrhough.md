---
title: "Walkthrough: Meethack x RomHack Cyber Car Wash CTF"
date: 2026-10-11T18:30:00
tags:
  - CTFs
  - RFID
  - Physical security
  - Hardware hacking

images:
- /images/meethack_ctf/tag_id.jpeg
- /images/meethack_ctf/no_keys.jpg
- /images/meethack_ctf/recovered_keys.jpeg
- /images/meethack_ctf/raw_content.jpeg
- /images/meethack_ctf/ascii_content.jpeg
- /images/meethack_ctf/ascii_content_2.jpeg
- /images/meethack_ctf/ascii_content_3.jpeg
- /images/meethack_ctf/ascii_content_4.jpeg
- /images/meethack_ctf/ascii_content_5.jpeg
- /images/meethack_ctf/ascii_content_6.jpeg
- /images/meethack_ctf/ascii_content_7.jpeg
- /images/meethack_ctf/ascii_content_8.jpeg
- /images/meethack_ctf/ascii_content_9.jpeg
- /images/meethack_ctf/read_data.png
- /images/meethack_ctf/forge_value.jpeg
- /images/meethack_ctf/value_block.jpeg
- /images/meethack_ctf/win.jpg
- /images/meethack_ctf/fetchkeys.png
- /images/meethack_ctf/endpoint_and_keys.png
- /images/meethack_ctf/fetchkeys_param.png
- /images/meethack_ctf/keys.png
- /images/meethack_ctf/backdoor.jpeg
- /images/meethack_ctf/backdoor_read.png
- /images/meethack_ctf/key_recovery.png

image:
- /images/meethack_ctf/no_keys.jpg

draft: true
---

Having moved to a new country recently, I've left behind some friends, including the `Meethack` Torino hacking group, and since they were organizing a CTF for their community stand at `RomHack`, I figured it would be nice to give it a try.
So, by knowing the right people I got early access to the CTF, so I setup  the necessary hardware as best as I could to try solving the challenge.

> The materials are now public, check out [here](https://meethack.it/resources/) if you want to give the challenge a try.

## Setup and tooling

I tried solving the challenge as I would on site: mostly using my phone; and to be fair I did most of the heavy lifting with it, however I could use the help of a tiny `ChameleonUltra` NFC tool. At some point I also used a `proxmark3`, but no spoilers. 

So the tools I used the most were the Chameleon and its companion app, as well as the `MIFARE Classic Tool` application, which came in handy in the past.

To play the CTF each user is given:

- NFC keyfob
- Cyber Car Wash APK file
- 18€ in credit loaded in the keyfob

With the objective to buy the most expensive item on the menu (5500€), through a point of sale device held by administrators.

The app only allows users to read their own tag, nothing more, nothing less, so I put it on the side for a minute and focused on something else.

## Poking at the NFC Tag

An obvious step to take is to read the values stored in the keyfob to better understand how to approach the challenge. Before proceeding further, it is necessary to identify the device.

|![tag_id](/images/meethack_ctf/tag_id.jpeg#center)|
| :----------------------------------: |
|            Missing keys - can't read sectors         |

We found that we are working with a MIFARE Classic 1k, however attempting to dump the content of the tag, we meet the first issue:

|![no keys](/images/meethack_ctf/no_keys.jpg#center)|
| :----------------------------------: |
|            Missing keys - can't read sectors         |

Only sector one is readable with default keys, all other sectors are protected by unknown keys.

At this point one can attempt doing multiple things, I decided to go for the hardware hacking way and attempt recovering the keys exploiting known key recovery attacks on MIFARE Classic 1k tags, namely Nested/Hard-nested and/or backdored tag.

> Mandatory disclaimer: since I played the CTF at home, I couldn't get my hands on the official keyfobs distributed at the conference. Instead I programmed some MIFARE Classic tags I had laying around. This means that the applicability of the key recovery attacks might vary depending on the actual tag you have in your hands.

## Getting the keys the unintended way 
In my case it was possible to recover the keys using the nested attack as the PRNG on my tag was not strong.
I simply used my `Chameleon` to do this, the attack is implemented in the app and it only takes a couple of minutes to retrieve all the keys.

|![recovered_keys](/images/meethack_ctf/recovered_keys.jpeg#center)|
| :----------------------------------: |
|            Keys recovered with nested attack         |

Having recovered the keys, we can dump the content of each sector, however it will look seemingly random.

|![raw_content](/images/meethack_ctf/raw_content.jpeg#center)|
| :----------------------------------: |
|            Raw content for keyfob sectors         |

But is it?

Decoding the sectors as ASCII, we notice a bunch of things:

|![ascii_content](/images/meethack_ctf/ascii_content.jpeg#center)|
| :----------------------------------: |
|            ASCII values from sectors        |

This is a good start, but what else do we have...

![ascii_content_2](/images/meethack_ctf/ascii_content_2.jpeg#center)

Prompt injection, fair play in my opinion, let's look for more stuff.

![ascii_content_3](/images/meethack_ctf/ascii_content_3.jpeg#center)

Now we can tell the creator of this challenge is a funny guy, what else?

![ascii_content_4](/images/meethack_ctf/ascii_content_4.jpeg#center)

Alright, alright, let's see where this goes.

![ascii_content_5](/images/meethack_ctf/ascii_content_5.jpeg#center)

\* drum roll \* 

![ascii_content_6](/images/meethack_ctf/ascii_content_6.jpeg#center)

Ok, this guy is really trying to be funny uh?

![ascii_content_7](/images/meethack_ctf/ascii_content_7.jpeg#center)

Just spill the beans already!

![ascii_content_8](/images/meethack_ctf/ascii_content_8.jpeg#center)

Alright, now we have a name if you want to take out your anger on somebody.

![ascii_content_9](/images/meethack_ctf/ascii_content_9.jpeg#center)

Alright, now violence might be the only option.

Well, not the kind of hint I was hoping for.


## Reversing the APK

Let's open the APK in `JADX` and see if we can go any further.

After looking around the code a little, we can find a bunch of useful functions in `com.meethack.romhack.camp2026.wrappers.NfcWrapper`: 

|![read_data](/images/meethack_ctf/read_data.png#center)|
| :----------------------------------: |
|            NFC related app functions         |

These read functions show us how data is stored into specific blocks. We can now start to figure out if and how we can forge data to put on the card.

To summarize what we can tell from these functsions:

- Name and surname are stored in ASCII in the blocks available in sector 14 (surname) and 15 (name).
- Creation date is stored in the first bytes of blocks 1,2 and 3 of sector 13, like so: year in block 0, month in block 1 and day in block 2.
- Amount is stored in block 3 as a 32 bit signed integer (little endian)

Knowing this we have a solid chance to actually be able to forge the amount value and attempt buying the most expensive item available on the point of sale.
So I used my chameleon to edit the dump and write it again on the keyfob.

Let's try reading back the card with forged data:

|![updated_val](/images/meethack_ctf/forge_value.jpeg#center)|
| :----------------------------------: |
|            First attempt at forging card amount         |

Sadly, when I tried purchasing using the point of sale app, I got no luck and received an error message in return.

> Missing screenshot notice: here I would have put a screenshot of the error, if I had one.

At this point I started wondering how the point of sale could detect errors, so I looked at the data a little closer and focused on diffs between two dumps, I started noticing a pattern, and finally I decided to check on the datasheet:

|![value_block](/images/meethack_ctf/value_block.jpeg#center)|
| :----------------------------------: |
|            MIFARE Value block documentation         |

Now it feels like the challenge is ready to be solved.

## Solving the CTF

We forge the data again paying attention to the value block using these functions:

```python
def make_value_block(value: int, block_address: int) -> bytes:
    """
    Create a MIFARE Classic value block.

    Layout:
        0x00-0x03  value       (LE)
        0x04-0x07  ~value
        0x08-0x0B  value       (LE)
        0x0C       block address
        0x0D       ~block address
        0x0E       block address
        0x0F       ~block address
    """

    # Unsigned 32-bit representation
    value_u32 = value & 0xffffffff

    value_bytes = struct.pack("<I", value_u32)
    inverse_value = bytes(b ^ 0xff for b in value_bytes)

    address = block_address & 0xff
    inverse_address = address ^ 0xff

    return (
        value_bytes + inverse_value + value_bytes +
        bytes([address, inverse_address, address,inverse_address])
    )

def make_string_blocks(value: str) -> list[bytes]:
    data = value.encode("utf-8")
    if len(data) > 48:
        raise ValueError(
            f"{value!r} is {len(data)} bytes; maximum is 48 bytes"
        )
    data = data.ljust(48, b"\x00")
    return [data[0:16], data[16:32], data[32:48]]

def forge_sector_12(amount: int) -> list[bytes]:
    return [
        make_value_block(amount, 48),
        bytes(16),
        bytes(16),
    ]

def forge_sector_13(date: datetime) -> list[bytes]:
    return [
        make_value_block(date.year, 52),
        make_value_block(date.month, 53),
        make_value_block(date.day, 54),
    ]

def forge_sector_14(surname: str) -> list[bytes]:
    return make_string_blocks(surname)

def forge_sector_15(name: str) -> list[bytes]:
    return make_string_blocks(name)
```

We write the edited content to the card, and when we try buying something from the point of sale app...

|![win](/images/meethack_ctf/win.jpg#center)|
| :----------------------------------: |
|            Challenge solved!         |

We finally got to buy the most expensive item and won the challenge.

## Getting the keys the intended way

To be completely fair, it didn't sit right with me that I used a vulnerability in my own keyfob to solve part of the challenge, so I dug deeper and not long after looking a bit more into the APK, I found where the app gets the keys:

|![fetchkeys](/images/meethack_ctf/fetchkeys.png#center)|
| :----------------------------------: |
|            fetch keys function         |

We see that it performs a POST request including an HTTP header and a JSON body containing the UID of the NFC tag. In the `/res/values/strings.xml` file we find the endpoint and API key header name and value.

|![endpoint_and_keys](/images/meethack_ctf/endpoint_and_keys.png#center)|
| :----------------------------------: |
|            String values for endpoint and keys         |

At this point I had blindly poked at the endpoint for a while and I could only get 502 errors. I then checked how fetch keys was called to better look at the UID value and there I noticed that the UID value has to be the entire `block 0` of the tag.

|![fetchkeys_param](/images/meethack_ctf/fetchkeys_param.png#center)|
| :----------------------------------: |
|            Parameter for fetchkeys         |

Now the endpoint started responding nicely with the keys for the tag:

|![keys](/images/meethack_ctf/keys.png#center)|
| :----------------------------------: |
|            Keys recovered the intended way         |

For convenience, here's the curl command I used:

```
curl -s -k -X POST "https://cyber-car-washer-alb-1288079676.eu-west-1.elb.amazonaws.com/" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "X-Api-Key: 1067d23605d6bd05aefa6ad0d0803b0a188ac33cfce932334d6ade20d6af02f07203382d27f5ad126ade76c0837e1319" \
  -d '{"uid":"0B55C2039F0804006263646566676869"}' | jq
```

## Getting the keys the unintended way pt. 2

By knowing the right people, I was able to get one of the original tags distributed at the CTF.

And so I got to work, I had yet to find a MIFARE Classic tag that effectively protects its keys.
Just like I had imagined, we have a new unintended solution to explore: abusing the `Fudan FM11RF08S` backdoor.

|![backdoor](/images/meethack_ctf/backdoor.jpeg#center)|
| :----------------------------------: |
|            Challenge keyfob is backdored        |

I had read about this issue, but never encountered it in the wild, so it was pretty fun to mess with too.

It's just a matter of reading the tag with known backdoor keys to see if any would work.
For reference these keys are:

- `A396EFA4E24F` (Mifare classic 1k)
- `A31667A8CEC1` (Mifare classic 1k)
- `518B3354E760` (Mifare classic 4k)

So I used my `proxmark3` to attempt using these keys to perform read operations

|![backdoor_read](/images/meethack_ctf/backdoor_read.png#center)|
| :----------------------------------: |
|            MIFARE keyfob read with backdoor key        |

However this doesn't give us write access, for the recovery of the keys, we'll just use a script in the in the `proxmark3` repo: `mf_backdoor_dump.py`

|![key_recovery](/images/meethack_ctf/key_recovery.png#center)|
| :----------------------------------: |
|            Backdoor based key recovery        |

This takes some time, between 15 and 30 minutes, but we demonstrated we could get the keys in another unintended way.

## Thanks

I wanted to thank the `Meethack` group for creating this challenge.
A special thank you to [IadRabbit](https://github.com/iadrabbit) for letting me try the challenge in advance and for keeping a keyfob on the side for me and to [\[kuomkuom\]](https://github.com/ohkuom) for bringing me the keyfob back from the conference.