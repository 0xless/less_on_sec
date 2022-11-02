---
title: "Walkthrough: BLE CTF"
date: 2022-04-17T14:08:32+02:00
toc: false
images:
tags:
  - CTFs
  - Radio
  - BLE 
---

This CTF has been in my todo list for a good while, finally I had the time to solve it and to publish a walkthrough article on it!
The challenge is available at: https://github.com/hackgnar/ble_ctf and is an awesome tool to learn about BLE and get your feet wet.

To be fair I expected the CTF to be a bit more security oriented, but I learned way more than I expected, both on BLE technologies and on the tools used to interact with those.

> ***Disclaimer***  
> I'm not a fan of walkthrough articles that ignore the request of the creators not to post those.
> The reasons I'm publishing this article are that:
>
> - No request by the creator has been made not to post the solutions
> - There are plenty of walkthrough articles on this CTF that don't really teach anything and barely explain the solutions
> - The CTF is self-hosted and is more of a teaching tool than a proper challenge
>
> Of course I strongly recommend trying to solve the challenges on your own before checking the solutions here! 
> Anyway I suggest reading the first part of the articles to gather some useful information about the technologies involved in the challenge.

Without further ado, let's get started!

## How does this CTF works?

The CTF is composed of a series of challenges hosted on an ESP32 board.
The software required to setup the challenge is in fact a firmware to be installed on the board. I won't enter in details about the installation in this article, but you can find detailed instructions here: https://github.com/hackgnar/ble_ctf/blob/master/docs/setup.md

When the firmware is installed, the ESP32 will host a GATT server with which we can interact to solve the challenges and gather the flags.

### What's a GATT server

GATT is the name of a protocol that is used to exchange data between two Bluetooth Low Energy devices.
This is developed on top of the Attribute Protocol (ATT) and manages device interactions following the advertising and pairing processes. 
A GATT Server is a device that stores attribute data locally and provides data access methods to a remote GATT Client paired via BLE.  A client, on the other hand, is a device that access data on a remote GATT Server. When two devices are paired, each can function as both a GATT Server and a GATT Client. 

GATT lists a device’s characteristics, descriptors, and services in a table as either 16 or 32-bits values. 
A characteristic is a data value sent between the server and the client. 

These characteristics can have descriptors that provide additional information about them. 
Characteristics are often grouped in services if they have purposes related to each other. Services can have several characteristics.

| ![GATT server layout](/images/BLE_CTF/gatt_server.png#center) |
| :----------------------------------------------------------: |
| Image Credits - Practical IoT Hacking: The Definitive Guide to Attacking the Internet of Things |

### ATT/GATT basics

It is possible to interact with a GATT server, by interacting with each of the characteristics available.
It is possible to do that using handles, or UUID: handles define the address of a characteristic, while the UUID works as an identifier, but also gives the user some indication about the content of a characteristic. In general, handles are to be preferred to reference to a particular characteristic, this is because UUID can vary depending on the GATT implementation, while handles are expected to remain immutable for each device. 

As anticipated, UUID contain info about the service or characteristic. These are useful because they are defined by profiles: standards that use known UUID to serve specific information, profiles can be found here: https://www.bluetooth.com/specifications/specs/
While UUID vary depending on the device functionality some UUID remains the same, for example: 0x2800, around which service discovery is built. This UUID allows GATT servers to understand service boundaries without having to know the exact standard the device is following.

Finally, there are characteristic properties. Properties indicate how to interact with a characteristic and what to expect from it.

| ![properties](/images/BLE_CTF/properties.png#center) |
| :----------------------------------------------------------: |
|            Image Credits - devzone.nordicsemi.com            |

Properties are defined as an hex value and work as bitmasks, so using bitwise operations it is possible reveal the properties of a characteristic starting from its properties value.

Properties need to be interpreted knowing Attribute Protocol Packets, these are:

- Commands         	&emsp;&ensp;&emsp;&emsp;(Client -> Server, no response required)

- Requests            	&emsp;&nbsp;&emsp;&emsp;&emsp;(Client -> Server, response required)

- Responses         	&emsp;&ensp;&emsp;&emsp;(Server -> Client, it's the response to a request)

- Notifications      	 &emsp;&ensp;&emsp;(Server -> Client, no response required - Signal the fact that a  

  &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp;    characteristic's value has changed)

- Indications        	&emsp; &nbsp;&emsp;&emsp;(Server -> Client, ACK response required by the client - Similar to   

  &emsp; &emsp; &emsp; &emsp; &emsp; &emsp;   &emsp; notifications)

- Confirmations   	&emsp; &nbsp;&ensp;(Client -> Server, it's the response to an indication)

As indicated in the image, the `Read` and `Write` properties are like permissions, and allow the reading and writing of the characteristic if set.
The other properties are an indication of the behavior of a characteristic in response of an action by the client.

Actions performed by the client are limited to read and write operations.
While read operations are requests by nature, write operations can either be commands or requests.

## CTF

The tools that will be employed for this challenge are: gatttool and hcitool.
There are other tools that can be used, in particular bettercap, which is a bit more user friendly but lacks some functionalities, or libraries that implement GATT operations, that were avoided to allow familiarizing with the tools available on a standard linux system.
I think this "Living off the Land" approach helps better understanding GATT operations and also leads to a more generally employable approach to solve BLE related tasks. Of course you're free to use the tools you prefer for this challenge.

Before starting, I suggest familiarizing with the challenge page and to understand how to operate actions relative to the challenge, like reading the score and submitting a flag. The list of the flags at the bottom of the page is really useful, as understanding exactly what to do can be tricky.
There are also hints there, don't be afraid to check those out. None of the hints gives away too much and in certain cases those are needed to understand the challenge and to get the flag.

NOTE: the score is reset each time the ESP32 is powered off! Keep that in mind.

---------------------

Let's begin! First thing first, we need to retrieve the device BT MAC address and we can do that using hcitool:

```
less@machine:~$ sudo hcitool lescan
LE Scan ...
78:21:84:80:A2:22 BLECTF
[...]
```

Now we know the address for my board is: `78:21:84:80:A2:22`, it will vary on yours.

Once we have the address we can start enumerating services on the GATT server.
The first thing to do is to connect to the board, we will do that using `gatttool` and its interactive mode:

```
less@machine:~$ gatttool -I
[                 ][LE]> connect 78:21:84:80:A2:22
Attempting to connect to 78:21:84:80:A2:22
Connection successful
```

Now we need to enumerate the services, we can do that using the `primary` command or by reading the UUID 0x2800.

```
[78:21:84:80:A2:22][LE]> primary
attr handle: 0x0001, end grp handle: 0x0005 uuid: 00001801-0000-1000-8000-00805f9b34fb
attr handle: 0x0014, end grp handle: 0x001c uuid: 00001800-0000-1000-8000-00805f9b34fb
attr handle: 0x0028, end grp handle: 0xffff uuid: 000000ff-0000-1000-8000-00805f9b34fb
[78:21:84:80:A2:22][LE]> char-read-uuid 0x2800
handle: 0x0001 	 value: 01 18 
handle: 0x0014 	 value: 00 18 
handle: 0x0028 	 value: ff 00 
```

And in both cases we retrieve services boundaries. The first two services are standard, and contain info about the server itself, while the third is the one we want to work with.

Now we can enumerate the characteristic of each service. This can be done using the command `characteristics` and using the boundaries found as upper and lower limits:

```
[78:21:84:80:A2:22][LE]> characteristics 0x0001 0x0014
handle: 0x0002, char properties: 0x20, char value handle: 0x0003, uuid: 00002a05-0000-1000-8000-00805f9b34fb
[78:21:84:80:A2:22][LE]> characteristics 0x0014 0x0028
handle: 0x0015, char properties: 0x02, char value handle: 0x0016, uuid: 00002a00-0000-1000-8000-00805f9b34fb
handle: 0x0017, char properties: 0x02, char value handle: 0x0018, uuid: 00002a01-0000-1000-8000-00805f9b34fb
handle: 0x0019, char properties: 0x02, char value handle: 0x001a, uuid: 00002aa6-0000-1000-8000-00805f9b34fb
[78:21:84:80:A2:22][LE]> characteristics 0x0028 0x00ff
handle: 0x0029, char properties: 0x02, char value handle: 0x002a, uuid: 0000ff01-0000-1000-8000-00805f9b34fb
handle: 0x002b, char properties: 0x0a, char value handle: 0x002c, uuid: 0000ff02-0000-1000-8000-00805f9b34fb
handle: 0x002d, char properties: 0x02, char value handle: 0x002e, uuid: 0000ff03-0000-1000-8000-00805f9b34fb
handle: 0x002f, char properties: 0x02, char value handle: 0x0030, uuid: 0000ff04-0000-1000-8000-00805f9b34fb
handle: 0x0031, char properties: 0x0a, char value handle: 0x0032, uuid: 0000ff05-0000-1000-8000-00805f9b34fb
handle: 0x0033, char properties: 0x0a, char value handle: 0x0034, uuid: 0000ff06-0000-1000-8000-00805f9b34fb
handle: 0x0035, char properties: 0x0a, char value handle: 0x0036, uuid: 0000ff07-0000-1000-8000-00805f9b34fb
handle: 0x0037, char properties: 0x02, char value handle: 0x0038, uuid: 0000ff08-0000-1000-8000-00805f9b34fb
handle: 0x0039, char properties: 0x08, char value handle: 0x003a, uuid: 0000ff09-0000-1000-8000-00805f9b34fb
handle: 0x003b, char properties: 0x0a, char value handle: 0x003c, uuid: 0000ff0a-0000-1000-8000-00805f9b34fb
handle: 0x003d, char properties: 0x02, char value handle: 0x003e, uuid: 0000ff0b-0000-1000-8000-00805f9b34fb
handle: 0x003f, char properties: 0x1a, char value handle: 0x0040, uuid: 0000ff0c-0000-1000-8000-00805f9b34fb
handle: 0x0041, char properties: 0x02, char value handle: 0x0042, uuid: 0000ff0d-0000-1000-8000-00805f9b34fb
handle: 0x0043, char properties: 0x2a, char value handle: 0x0044, uuid: 0000ff0e-0000-1000-8000-00805f9b34fb
handle: 0x0045, char properties: 0x1a, char value handle: 0x0046, uuid: 0000ff0f-0000-1000-8000-00805f9b34fb
handle: 0x0047, char properties: 0x02, char value handle: 0x0048, uuid: 0000ff10-0000-1000-8000-00805f9b34fb
handle: 0x0049, char properties: 0x2a, char value handle: 0x004a, uuid: 0000ff11-0000-1000-8000-00805f9b34fb
handle: 0x004b, char properties: 0x02, char value handle: 0x004c, uuid: 0000ff12-0000-1000-8000-00805f9b34fb
handle: 0x004d, char properties: 0x02, char value handle: 0x004e, uuid: 0000ff13-0000-1000-8000-00805f9b34fb
handle: 0x004f, char properties: 0x0a, char value handle: 0x0050, uuid: 0000ff14-0000-1000-8000-00805f9b34fb
handle: 0x0051, char properties: 0x0a, char value handle: 0x0052, uuid: 0000ff15-0000-1000-8000-00805f9b34fb
handle: 0x0053, char properties: 0x9b, char value handle: 0x0054, uuid: 0000ff16-0000-1000-8000-00805f9b34fb
handle: 0x0055, char properties: 0x02, char value handle: 0x0056, uuid: 0000ff17-0000-1000-8000-00805f9b34fb
```

And just like that we retrieved every characteristic available, their value and their properties.

At this point it is useful to define utility functions to work with the handles:

```bash
mac="78:21:84:80:A2:22" #change the value to match your board's MAC

# HEX encode
encode() {
        echo -n $1 | xxd -ps;
}

# HEX decode
decode() {
        echo $1 | tr -d "" | xxd -r -p; echo
}

# Write flag to handle 0x002c
submit_flag() {
        gatttool -b $mac --char-write-req -a 0x002c -n `encode "$1"`
}

# Read handle 0x002a to get the score
get_score() {
        read_hnd 0x002a
}

# Read values from handle and decode it
read_hnd() {
        decode "`gatttool -b $mac --char-read -a $1 | awk -F':' '{print $2}'`"
}

# List properties from hex property value
get_properties() {
		if (( ($1 & 0x1) == 0x1 )); then echo "Broadcast"; fi
        if (( ($1 & 0x2) == 0x2 )); then echo "Read"; fi
        if (( ($1 & 0x4) == 0x4 )); then echo "Write without response"; fi
        if (( ($1 & 0x8) == 0x8 )); then echo "Write"; fi
        if (( ($1 & 0x10) == 0x10 )); then echo "Notify"; fi
        if (( ($1 & 0x20) == 0x20 )); then echo "Indicate"; fi
        if (( ($1 & 0x40) == 0x40 )); then echo "Authenticated Signed Writes"; fi
        if (( ($1 & 0x80) == 0x80 )); then echo "Extended Properties"; fi
}
```

NOTE: properties for each characteristic of interest are listed in the challenge description, so the `get_properties` function won't be used explicitly. 

-----------------

- **Flag #1** 
  To get this flag we need to use the [hint](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag1.md)! 

```
less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req -a 0x002c -n $(echo -n "12345678901234567890"|xxd -ps)
Characteristic value was written successfully
```

Now, reading the handle 0x002a confirms we got the first flag.

```
less@machine:~$ get_score
Score:1 /20
```

- **Flag 0x002e**

  >  Properties: Read

  To get this flag we simply need to read the handle 0x002e.

  ```
  less@machine:~$ read_hnd 0x002e
  d205303e099ceff44835
  less@machine:~$ submit_flag d205303e099ceff44835
  less@machine:~$ get_score
  Score:2 /20
  ```

- **Flag 0x0030**

  > "MD5 of Device Name"
  >
  >  Properties: Read

  So for this flag we need to submit the MD5 of the Device Name.
  Note: the MD5 hash needs to be truncated to 20 characters as suggested in the README of the CTF project.

  We already know the device name as we encountered it connecting to the device:

  ```
  less@machine:~$ sudo hcitool lescan
  LE Scan ...
  78:21:84:80:A2:22 BLECTF
  [...]
  ```

  Now we only need to get the MD5 digest of the name, cut and submit it:

  ```
  less@machine:~$ echo -n "BLECTF" | md5sum | cut -c 1-20
  5cd56d74049ae40f442e
  less@machine:~$ submit_flag 5cd56d74049ae40f442e
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:3 /20
  ```

- **Flag 0x0016**

  This is a particular challenge, the hint says: 

  > "Bluetooth GATT services provide some extra device attributes.  Try finding the value of the Generic Access -> Device Name."

  But the name of the challenge is "Flag 0x0016", so it gives it away!

  Reading the handle we get the flag:

  ```
  less@machine:~$ read_hnd 0x0016
  2b00042f7481c7b056c4b410d28f33cf
  ```

  Of course we need to cut it before submitting it (as it is a name and the README clearly states that names and digests need to be cut to the 20th character):

  ```
  less@machine:~$ echo 2b00042f7481c7b056c4b410d28f33cf | cut -c 1-20
  2b00042f7481c7b056c4
  less@machine:~$  submit_flag 2b00042f7481c7b056c4
  Characteristic value was written successfully
  less@machine:~$  get_score 
  Score:4 /20
  ```

  But it is important to understand the reason why this flag is there.
  We already talked about profiles and standard UUID, these include an UUID that corresponds to the Device Name, this UUID is: 0x2A00.

  Using `gatttool`, we can read the UUID and note that the handle is indeed 0x0016:

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-read -u 0x2A00
  handle: 0x0016 	 value: 32 62 30 30 30 34 32 66 37 34 38 31 63 37 62 30 35 36 63 
  ```

- **Flag 0x0032**

  > The content of the handle is: "Write anything here"
  >
  >  Properties: Read, Write

  To get the flag we need to write "anything" to the 0x0032 handle.

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req -a 0x0032 -n `encode "anything"`
  Characteristic value was written successfully
  ```

  Now, we can read the handle a second time and it will reveal the flag

  ```
  less@machine:~$ read_hnd 0x0032
  3873c0270763568cf7aa
  less@machine:~$ submit_flag 3873c0270763568cf7aa
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:5 /20
  ```

- **Flag 0x0034**

  > The content of the handle is: "Write the ascii value "yo" here"
  >
  > Properties: Read, Write

  The task is similar to the last one, but this time we don't have to hex encode the string

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req -a 0x0034 -n `encode "yo"`
  Characteristic value was written successfully
  ```

  Now, we can get the flag reading the handle again

  ```
  less@machine:~$ read_hnd 0x0034
  c55c6314b3db0a6128af
  less@machine:~$ submit_flag c55c6314b3db0a6128af
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:6 /20
  ```

- **Flag 0x0036**

  > The content of the handle is: "Write the hex value 0x07 here"
  >
  > Properties: Read, Write

  This task is not different from the previous ones, but since the default encoding used in BLE communications is hex, and the value is to be sent in hex, no encoding needs to be used

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req -a 0x0036 -n 07
  Characteristic value was written successfully
  ```

  Now the flag should be available at the same handle

  ```
  less@machine:~$ read_hnd 0x0036
  1179080b29f8da16ad66
  less@machine:~$ submit_flag 1179080b29f8da16ad66
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:7 /20
  ```

- **Flag 0x0038**

  > The content of the handle is: "Write 0xC9 to handle 58"
  >
  > Properties: Read
  > Handle 58 properties: Write

  First off we need to convert 58 in hex to get the hex value for the handle, `hex(58)` is 0x3A.
  Now we need to write the hex value 0xC9 to the handle, this can be done exactly like we did for the last challenge:

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req -a 0x003A -n C9
  Characteristic value was written successfully
  ```

  Now reading the handle 0x0038 again reveals the flag

  ```
  less@machine:~$ read_hnd 0x0038
  f8b136d937fad6a2be9f
  less@machine:~$ submit_flag f8b136d937fad6a2be9f
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:8 /20
  ```

- **Flag 0x003C**

  > The content of the handle is: "Brute force my value 00 to ff"
  >
  > Properties: Read, Write

  This can be achieved in multiple ways, but for consistency `gatttool` and a bit of bash scripting will be used.

  ```
  less@machine:~$ for i in {0..255};
  > do gatttool -b 78:21:84:80:A2:22 --char-write-req -a 0x003c -n $(printf '%02x\n' $i) > /dev/null;
  > done;
  ```

  This script iterates through every value from 0 to 255 (which is FF in hex) and writes such value in the handle 0x003C. The `$(printf '%02x\n' $i);` command converts the value to write from decimal to hex.

  Once the script has finished, it's possible to read the handle 0x003C again to get the flag

  ```
  less@machine:~$ read_hnd 0x003c
  933c1fcfa8ed52d2ec05
  less@machine:~$ submit_flag 933c1fcfa8ed52d2ec05
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:9 /20
  ```

- **Flag 0x003E**

  > The content of the handle is: "Read me 1000 times"
  >
  > Properties: Read

  This can be done simply modifying the script used in the previous challenge

  ```
  less@machine:~$ for i in {0..1000}; 
  > do gatttool -b 78:21:84:80:A2:22 --char-read -a 0x003e;
  > done;
  ```

  This command will print out the content of the reads, and after a while you'll notice that the output changes like in the example here:

  ```
  [...]
  Characteristic value/descriptor: 52 65 61 64 20 6d 65 20 31 30 30 30 20 74 69 6d 65 73 
  Characteristic value/descriptor: 52 65 61 64 20 6d 65 20 31 30 30 30 20 74 69 6d 65 73 
  Characteristic value/descriptor: 52 65 61 64 20 6d 65 20 31 30 30 30 20 74 69 6d 65 73 
  Characteristic value/descriptor: 36 66 66 63 64 32 31 34 66 66 65 62 64 63 30 64 30 36 39 65 
  Characteristic value/descriptor: 36 66 66 63 64 32 31 34 66 66 65 62 64 63 30 64 30 36 39 65 
  Characteristic value/descriptor: 36 66 66 63 64 32 31 34 66 66 65 62 64 63 30 64 30 36 39 65 
  ```

  The values printed after the variation represent the flag!

  So we just need to submit it to complete this challenge

  ```
  less@machine:~$ decode "36 66 66 63 64 32 31 34 66 66 65 62 64 63 30 64 30 36 39 65"
  6ffcd214ffebdc0d069e
  less@machine:~$ submit_flag 6ffcd214ffebdc0d069e
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:10/20 
  ```

- **Flag 0x0040**

  > The content of the handle is: "Listen to me for a single notification"
  >
  > Properties: Read, Write, Notify

  To listen for a notification, we first need to send a write request, but we will have to listen to the notification, this can be achieved with the following command:

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req --handle=0x0040 --value=dummy --listen
  Characteristic value was written successfully
  Notification handle = 0x0040 value: 35 65 63 33 37 37 32 62 63 64 30 30 63 66 30 36 64 38 65 62 
  ```

  The notification value is the flag!

  ```
  less@machine:~$ decode "35 65 63 33 37 37 32 62 63 64 30 30 63 66 30 36 64 38 65 62"
  5ec3772bcd00cf06d8eb
  less@machine:~$ submit_flag 5ec3772bcd00cf06d8eb
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:11/20 
  ```

- **Flag 0x0042**

  > The content of the handle is: "Listen to handle 0x0044 for a single indication"
  >
  > Properties: Read
  >
  > Handle 0x0044 Properties: Read, Write, Indicate

  This flag can be obtained just like the last one:

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req --handle=0x0044 --value=ffff --listen
  Characteristic value was written successfully
  Indication   handle = 0x0044 value: 63 37 62 38 36 64 64 31 32 31 38 34 38 63 37 37 63 31 31 33 
  ```

  Once the notification value is retrieved we can decode and submit it:

  ```
  less@machine:~$ decode "63 37 62 38 36 64 64 31 32 31 38 34 38 63 37 37 63 31 31 33"
  c7b86dd121848c77c113
  less@machine:~$ submit_flag c7b86dd121848c77c113
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:12/20 
  ```

- **Flag 0x0046**

  > The content of the handle is: "Listen to me for multi notifications"
  >
  > Properties: Read, Write, Notify

  And with the same command we can get this flag too!

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req --handle=0x0046 --value=ffff --listen
  Characteristic value was written successfully
  Notification handle = 0x0046 value: 55 20 6e 6f 20 77 61 6e 74 20 74 68 69 73 20 6d 73 67 00 00 
  Notification handle = 0x0046 value: 63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64 
  Notification handle = 0x0046 value: 63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64 
  Notification handle = 0x0046 value: 63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64 
  ```

  After the first notification, we get the content of the flag! (The decoded content of the first notification is "U no want this msg").

  ```
  less@machine:~$ decode "63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64"
  c9457de5fd8cafe349fd
  less@machine:~$ submit_flag c9457de5fd8cafe349fd
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:13/20 
  ```

- **Flag 0x0048**

  > The content of the handle is: "Listen to handle 0x004a for multi indications"
  >
  > Properties: Read
  >
  > Handle 0x004a Properties: Read, Write, Indicate

  Once again the same command is the key to the flag!

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req --handle=0x004a --value=ffff --listen
  Characteristic value was written successfully
  Indication   handle = 0x004a value: 55 20 6e 6f 20 77 61 6e 74 20 74 68 69 73 20 6d 73 67 00 00 
  Indication   handle = 0x004a value: 62 36 66 33 61 34 37 66 32 30 37 64 33 38 65 31 36 66 66 61 
  Indication   handle = 0x004a value: 62 36 66 33 61 34 37 66 32 30 37 64 33 38 65 31 36 66 66 61 
  Indication   handle = 0x004a value: 62 36 66 33 61 34 37 66 32 30 37 64 33 38 65 31 36 66 66 61 
  ```

  Now we can decode and submit the flag like always.

  ```
  less@machine:~$ decode "62 36 66 33 61 34 37 66 32 30 37 64 33 38 65 31 36 66 66 61"
  b6f3a47f207d38e16ffa
  less@machine:~$ submit_flag b6f3a47f207d38e16ffa
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:14/20 
  ```

- **Flag 0x004c**

  > The content of the handle is: "Connect with BT MAC address 11:22:33:44:55:66"
  >
  > Properties: Read

  Now, this challenge depends on the BT card you're using. I'm using a raspberry pi 3b+ and I could solve the challenge in the following way:

  ```
  less@machine:~$ sudo hcitool cmd 0x3f 0x001 0x66 0x55 0x44 0x33 0x22 0x11
  < HCI Command: ogf 0x3f, ocf 0x0001, plen 6
    66 55 44 33 22 11 
  > HCI Event: 0x0e plen 4
    01 01 FC 00 
  less@machine:~$ sudo hciconfig hci0 reset
  less@machine:~$ systemctl restart bluetooth.service
  Authentication is required to restart 'bluetooth.service'.
  Authenticating as: ,,, (less)
  Password: 
  ==== AUTHENTICATION COMPLETE ===
  ```

  At this point we can confirm our BT MAC address executing `hciconfig`:

  ```
  less@machine:~$ hciconfig
  hci0:	Type: Primary  Bus: UART
  	BD Address: 11:22:33:44:55:66  ACL MTU: 1021:8  SCO MTU: 64:1
  	UP RUNNING 
  	RX bytes:13820 acl:117 sco:0 events:824 errors:0
  	TX bytes:16604 acl:108 sco:0 commands:606 errors:0
  ```

  And of course now, reading the handle gives us the flag:

  ```
  less@machine:~$ read_hnd 0x004c
  aca16920583e42bdcf5f
  less@machine:~$ submit_flag aca16920583e42bdcf5f
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:15/20
  ```

  Check out these resources for more info on this solution: 
  https://www.lisha.ufsc.br/teaching/shi/ine5346-2003-1/work/bluetooth/hci_commands.html and https://raspberrypi.stackexchange.com/a/124117

- **Flag 0x004e**

  > The content of the handle is: "Set your connection MTU to 444" 
  >
  > Properties: Read

  Even if there is the option to do that (-m), I can't seem to get the flag without using the interactive mode of gatttools, so my solution to this flag is:

  ```
  less@machine:~$ gatttool -I
  [                 ][LE]> connect 78:21:84:80:A2:22
  Attempting to connect to 78:21:84:80:A2:22
  Connection successful
  [78:21:84:80:A2:22][LE]> mtu 444
  MTU was exchanged successfully: 444
  [78:21:84:80:A2:22][LE]> char-read-hnd 0x004e
  Characteristic value/descriptor: 62 31 65 34 30 39 65 35 61 34 65 61 66 39 66 65 35 31 35 38 
  [78:21:84:80:A2:22][LE]> exit
  ```

  And just like that we got the flag

  ```
  less@machine:~$ decode "62 31 65 34 30 39 65 35 61 34 65 61 66 39 66 65 35 31 35 38 "
  b1e409e5a4eaf9fe5158
  less@machine:~$ submit_flag b1e409e5a4eaf9fe5158
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:16/20
  ```

- **Flag 0x0050**

  > The content of the handle is: "Write+resp 'hello' " 
  >
  > Properties: Read, Write

  I suggest checking the hint for this challenge! The key is in correctly handling the ACK response from the write instruction, this is the reason

  `--char-write-req` needs to be used. 

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req --handle=0x0050 --value=`encode hello`
  Characteristic value was written successfully
  less@machine:~$ read_hnd 0x0050
  d41d8cd98f00b204e980
  ```

  Now we only need to submit the flag.

  ```
  less@machine:~$ submit_flag d41d8cd98f00b204e980
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:17/20
  ```

- **Flag 0x0052**

  > The content of the handle is: "No notifications here! really?"
  >
  > Properties: Read, Write

  In fact even if there is no notification property set...

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req --handle=0x0052 --value=`encode hello` --listen
  Characteristic value was written successfully
  Notification handle = 0x0052 value: 66 63 39 32 30 63 36 38 62 36 30 30 36 31 36 39 34 37 37 62 
  ```

  The flag is revealed through a notification!
  Now we can decode and submit it.

  ```
  less@machine:~$ decode "66 63 39 32 30 63 36 38 62 36 30 30 36 31 36 39 34 37 37 62"
  fc920c68b6006169477b
  less@machine:~$ submit_flag fc920c68b6006169477b
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:18/20
  ```

- **Flag 0x0054**

  > The content of the handle is: "So many properties!" 
  >
  > Properties: Broadcast, Read, Write, Notify, Extended properties

  ...and to be fair this handle has "Broadcast, Read, Write, Notify, Extended properties" properties. It's a bit of a mess!

  But getting the flag is fairly simple. The first half can be retrieved simply listening for a notification:

  ```
  less@machine:~$ gatttool -b 78:21:84:80:A2:22 --char-write-req --handle=0x0054 --value=ffff --listen
  Characteristic value was written successfully
  Notification handle = 0x0054 value: 30 37 65 34 61 30 63 63 34 38 
  ```

  For the second half we need to read the handle once more:

  ```
  less@machine:~$ read_hnd 0x0054
  fbb966958f
  ```

  Now we just need to put the two halves together and submit the flag (the second half of the flag goes first!):

  ```
  less@machine:~$ submit_flag fbb966958f07e4a0cc48
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:19/20
  ```

- **Flag 0x0056**

  > The content of the handle is: "md5 of author's twitter handle".
  >
  > Properties: Read

  So we need to do a bit of OSINT to solve this challenge!

  The starting point is the github page of the challenge: https://github.com/hackgnar/ble_ctf
  Luckily in the README.md file we can find a twitter follow button, this leads us to: https://twitter.com/hackgnar
  And now we know the handle is: @hackgnar

  Now we can get the MD5 digest of the handle and cut it to 20 chars:

  ```
  less@machine:~$ echo -n "@hackgnar" | md5sum | cut -c 1-20
  d953bfb9846acc2e15ee
  ```

  We can now submit it as a flag

  ```
  less@machine:~$ submit_flag d953bfb9846acc2e15ee
  Characteristic value was written successfully
  less@machine:~$ get_score 
  Score:20/20
  ```


------------

## Conclusions

Hope this article is useful to anyone who's stuck on one of these challenges or to anyone who's trying to learn about BLE technologies.
Writing this article has helped me formalizing most of the things I learned solving these challenges and my main takeaways are an improved understanding of BLE, GATT and ATT, but also the ability to use the tools needed to work with these technologies.

In future I'll try working with GATT libraries to programmatically interact with GATT servers!

---------------

Resources: 

- https://epxx.co/artigos/bluetooth_gatt.html (ATT/GATT basics)
- Practical IoT Hacking: The Definitive Guide to Attacking the Internet of Things (General BLE introduction)
- https://axodyne.com/2020/08/ble-uuids/  (Explanation on standard UUIDS)
- https://learn.adafruit.com/introduction-to-bluetooth-low-energy/gatt (Profiles and services)
- https://devzone.nordicsemi.com/guides/short-range-guides/b/bluetooth-low-energy/posts/ble-characteristics-a-beginners-tutorial (Properties)

