---
title: "Email spoofing 101"
date: 2022-02-05T16:40:50+01:00
tags:
  - DNS
  - Email

---

Email spoofing is a technique mostly linked to malicious activities including phishing and spamming.
From a formal(ish) standpoint, email spoofing is the act of sending an email with a forged sender address.

I got interested in this technique in the last few days, so I decided to experiment a bit with it.

## Email spoofing basics

This technique is as old as emails are, in fact original transmission protocols (such as SMTP) don't have authentication methods, and while security measures have been adopted these are [not widely employed yet](https://labs.bishopfox.com/tech-blog/2017/05/how-we-can-stop-email-spoofing).

An email is composed of three different parts:

- **Envelope**: data about sender and receiver addresses. 
  This part of message is destined to the server, it's generally not shown to the user by the client.
- **Header**: meta-data of the email. Contains information like the subject of the email, the date and some info about sending/receiving addresses.
- **Body**: the content of the email.

To get a bit more into details, the SMTP protocol specify two addresses in the envelope addressing part of the message:

- `MAIL FROM`: it specifies the "return address" in case an email bounces.
- `RCPT TO`: address to which the email will be delivered.

Other headers are often present, this set of information represent what the receiver will see when reading the email inside of a client. 
The addresses in each header will be present in the form of: `Foo Bar <foobar@spam.com>`.

In particular, these headers are: `From:`, `Reply-to:`, `Sender:`. Each of these headers represents the address visible to the receiving part.

In general, it's important to notice the difference between the envelope `MAIL FROM` or [RFC5321](https://datatracker.ietf.org/doc/html/rfc5321).MailFrom and the header `From:`or [RFC5322](https://datatracker.ietf.org/doc/html/rfc5322).From since the latter is shown to the user and can be easily spoofed and it's shown to the recipient of the email by the client.

As anticipated, SMTP doesn't check whether an user is authorized or not to use a particular address in these fields and this is the reason why it's so easy to spoof emails. The only reliable information we can harvest from a spoofed email is the IP from which the email was sent.

## Available security measures

There are three main security measured against email spoofing:

- **[Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208) (SPF)**
  This method allows the detection of forged sender addresses and it's based on envelope information.
  In particular SPF information are contained inside a `TXT` DNS record. These info specify which IP addresses are allowed to send emails on the behalf of the  domain. Allowed senders can be listed explicitly specifying authorized IP ranges, or checking if `A`, `AAAA` or `MX` records resolve to the sender's address.

  SPF also use 4 qualifiers, these represent the status of the SPF validation.
  Qualifiers can be:

  - PASS (+) : if the check is passed. The email can be accepted.
  - NEUTRAL (?) : if no policy is in use.
  - FAIL (-) : if the check wasn't successful. The email should be rejected.
  - SOFTFAIL (~) : like fail, but used for debugging. In general emails with SOFTFAIL qualifier are accepted but flagged.

  The SPF `TXT` DNS record is composed of SPF version (specified with the tag `v=` [it can only be followed by the value `spf1`](http://www.open-spf.org/SPF_vs_Sender_ID/#0.1.3)) and one of more *mechanisms*.
  Mechanisms are used to specify if a domain can send the email. Mechanisms include specifying IPv4/IPv6 address ranges or checking if A, MX or PTR records resolve to the domain. Other mechanisms exist but are rarely used.

  SPF alone is not enough to detect email forging because it checks envelope information only, it would still be possible to keep the original sender in the envelope information and forge only the headers that are interpreted by the client and can't be checked with SPF. 
  Usually a failed SPF check is enough to mark the email as spam or even reject it.
  It's worth noting that SPF policies could cause troubles in case of email forwarding even if the sender is authorized. That's because the "forwarder" is technically not authorized to send emails on the behalf the original domain.

- **[DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376/) (DKIM)**
  DKIM is an authentication method that allows the receiver to check if the email was authorized by the owner of the domain and functions as an integrity check to make sure the message hasn't been modified. 
  This check is possible thanks to an additional header containing a digital signature. A public key, that makes the signature validation possible, is available in a dedicated `TXT` DNS record. This method only checks the validity of email's header and body, the envelope addresses are ignored. It's possible to specify which headers to check, but the `RFC5322.From` header must always be checked.
  DKIM checks can and should be done at every SMTP hop and should resist relaying across [MTAs](https://en.wikipedia.org/wiki/Message_transfer_agent) .

  Important info contained in the DKIM headers of an email are:

  - `b`: signature of header and body
  - `bh`: body hash
  - `d`: signing domain
  - `s`: selector
  - `h`: header fields that has been signed

  But information about the protocol, the algorithm used to sign and more can be present in optional tags.

  The information useful to verify the signature are present in the `TXT` DNS record and are presented in the form of key-value pairs:

  - `k`: algorithm to be used
  - `v`: version of DKIM
  - `p`: public key to validate the signature

  More tags can be present, but are not useful for our analysis.

  DKIM records are published at `selector._domainkey.example.com`.

- **[Domain-based Message Authentication, Reporting and Conformance](https://datatracker.ietf.org/doc/html/rfc7489) (DMARC)**
  DMARC is an email authentication protocol. Using a `TXT` DNS record, it dictates the conditions under which an email is considered authenticated.
  It works on top of DKIM and SPF: these two mechanisms, as shown above, offer different checks on the authenticity of an email sender.  
  Both DKIM and SPF can offer authenticated identifiers in DMARC, meaning that DKIM `d=` tag of a validated DKIM-Signature header field and SPF validated `MAIL FROM` field can be used to validate an email's sender.
  
  DMARC alignment basically checks that the domain name in the `RFC5322.From` field, is "aligned" (matches) with either DKIM or SPF authenticated identifier.
  There are two types of alignment in DMARC, *strict* or *relaxed*.
  With strict alignment, the domain needs to be identical, while with relaxed alignment, a domain is considered aligned if the top-level domain match.
  
  It's also possible to define policies to specify how to deal with authentication failures.
  These policies are:
  
  - `none`: no action required by the receivers. It allows the domain to receive feedback reports.
  - `quarantine`: suggests that emails that fail DMARC check are flagged (marked as spam).
  - `reject`: suggests that emails that fail DMARC check are discarded.
  
  A `TXT` DNS entry for DMARC is composed of:
  
  - `v`: version 
  - `p`: policy
  - `sp`: sub-domain policy
  - `pct`: percentage of non authenticated emails to which apply the policy
  - `rua`: email address to send reports to 
  
  The use of `pct` is interesting because it can lower the protection offered by DMARC.
  By default `pct` is set to 100%, but if it wasn't, the receiver should use a Bernoulli sampling algorithm to check how the policy is to be applied to the email.
  If an email doesn't fall under the percentage specified in `pct`, a lower policy is to be applied to the email.
  So if `p=reject`, the quarantine policy is to be used and if `p=quarantine`, the `none` policy is to be applied.
  
  DMARC records are published at `_dmarc.example.com`.

While SPF and DKIM work properly, they both have defects:

- SPF checks the envelope from address, leaving the header from address not considered.
  This means that a malicious user can send an email from a whitelisted domain and spoof the header from address. This can lead to a situation in which the email is considered valid, but a domain different than the expected one is shown to the receiver.
- DKIM only checks parts of the email, this means that, leaving such parts untouched, a malicious user can [add other headers](https://www.zdnet.com/article/dkim-useless-or-just-disappointing/) and make the receiving part see an apparently valid message, but with spoofed parts.

This leaves DMARC as the only viable solution for a more comprehensive domain security.

 ## Vulnerable configurations

When it comes to email security implemented with DMARC, there are 4 main vulnerable scenarios to consider:

- Case 1  
  There is a DMARC DNS record, this entry describes a DMARC entry and it has a policy different than "none".
  In this case it's impossible to spoof an email from this domain.
- Case 2  
  There is a DMARC DNS record and a valid DMARC entry, but the policy is set to "none", meaning that, even if feedback reports will be sent, the domain is spoofable in emails.
- Case 3  
  There is no DMARC DNS record in a domain that is used to send emails. This is the worst possible scenario because the domain is completely unprotected.
- Case 4  
  There is no DMARC DNS record in a domain not used to send emails. The domain is spoofable, but the impact is not as big as in case 3.
  It is suggested to have DMARC in place even for domain not used to send emails, but it's not fundamental.

Of course to thoroughly evaluate the security in place, it's important to consider eventual SPF and DKIM policies, even if no DMARC protection is set up.

The most direct way to check the configuration is to query the DNS, for this article I'll specify the way to do this in a unix based system, but more info can be found [here](https://www.cisco.com/c/en/us/support/docs/security/secure-email-gateway/217073-how-to-use-dig-nslookup-to-find-spf-dki.html). We will use google.com as a domain for our experiments.

### Find the SPF record

Using `dig` it's possible to get `TXT` records for a domain.
The command is `dig google.com txt` and the output looks like this:

```
; <<>> DiG 9.16.8-Ubuntu <<>> google.com txt
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8353
;; flags: qr rd ra; QUERY: 1, ANSWER: 9, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;google.com.			IN	TXT

;; ANSWER SECTION:
google.com.		3600	IN	TXT	"MS=E4A68B9AB2BB9670BCE15412F62916164C0B20BB"
google.com.		3600	IN	TXT	"v=spf1 include:_spf.google.com ~all"
google.com.		3600	IN	TXT	"google-site-verification=wD8N7i1JTNTkezJ49swvWW48f8_9xveREV4oB-0Hf5o"
google.com.		3600	IN	TXT	"google-site-verification=TV9-DBe4R80X4v0M4U_bd_J9cpOJM0nikft0jAgjmsQ"
google.com.		3600	IN	TXT	"apple-domain-verification=30afIBcvSuDV2PLX"
google.com.		3600	IN	TXT	"docusign=05958488-4752-4ef2-95eb-aa7ba8a3bd0e"
google.com.		3600	IN	TXT	"globalsign-smime-dv=CDYX+XFHUw2wml6/Gb8+59BsH31KzUr6c1l2BPvqKX8="
google.com.		3600	IN	TXT	"facebook-domain-verification=22rm551cu4k0ab0bxsw536tlds4h95"
google.com.		3600	IN	TXT	"docusign=1b0a6754-49b1-4db5-8540-d2c12664b289"

;; Query time: 328 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Tue Nov 23 22:29:59 CET 2021
;; MSG SIZE  rcvd: 625
```

then we need to find the SPF record between `TXT` records. We simply need to find the one starting with `v=spf`, in this case it is 
`google.com.  3600	IN	TXT	"v=spf1 include:_spf.google.com ~all"`. Or we can use `grep` to extract the information directly: 
`dig google.com txt | grep spf`.

### Find the DKIM record

The DKIM entry is located at `selector._domainkey.example.com`, so to find the DKIM record we first need to find the DKIM selector.
This can be done in two ways, get it from an email received from the domain we are researching or trying to achieve a zone transfer.

From an email, we just need to look for the original message (containing the headers), if DKIM is enabled, one of the fields will be `DKIM-Signature:` followed by different elements; what we are looking for is `s=` that is followed by the selector.
If we can't receive an email from a specific domain, it's possible to try and get a zone transfer, it's basically a dump of every DNS record from a nameserver.
This can be done with the command: `dig axfr @nameserver example.com`.  The nameservers can be found using the command `dig ns example.com`.
Sadly google.com won't allow zone transfer. It's possible to practice this technique using [zonetransfer.me](https://digi.ninja/projects/zonetransferme.php), this website also contains a lot of cool info about DNS and zone transfers.

I was able to recover the selector from an email received from google.com. Once we have the selector, we can get info about DKIM using the command: 
`dig 20210112._domainkey.google.com txt`.
The output looks like this: 

```
; <<>> DiG 9.16.8-Ubuntu <<>> 20210112._domainkey.google.com txt
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52118
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;20210112._domainkey.google.com.	IN	TXT

;; ANSWER SECTION:
20210112._domainkey.google.com.	3600 IN	TXT	"v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtnbQAeqvEP2pG2L540W9JvVU7qy767Zs83Hjw34PCkZ9/9dWE6cK+CYaMNIQqTGfwq4uUe3diBuz3Uikkr+WPMj9AuxtxJqUAO8PsZ2Y5DneFpz5kVesC9/rtXgCwgYOmO" "9UjSy4IN11ewXUBuCH+zp2v5Abv5T0Lol/nWl8wLgRI1IesstingY4cnSfo3Pq3U0I1GAxdNFCK2FPedPpg4sdPpHqtxVvRLMLamRKoUfySBMjpXuMNL0UeCizmFfdUL73ZdiS+MNxGECzFNmeCngFBscLQN++urvfB9OqHQrbxLIwNyni3KMbE/E+cxPkx4KxhyGHSU2klV1vvIAHfwIDAQAB"

;; Query time: 92 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Tue Nov 23 22:23:32 CET 2021
;; MSG SIZE  rcvd: 483
```

### Find the DMARC record

The DMARC record is easy to find, just for the SPF record, we can use `dig _dmarc.google.com txt` to get what we are looking for.

The output is:

```
; <<>> DiG 9.16.8-Ubuntu <<>> _dmarc.google.com txt
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28825
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;_dmarc.google.com.		IN	TXT

;; ANSWER SECTION:
_dmarc.google.com.	300	IN	TXT	"v=DMARC1; p=reject; rua=mailto:mailauth-reports@google.com"

;; Query time: 44 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Tue Nov 23 22:29:14 CET 2021
;; MSG SIZE  rcvd: 117
```

Another way to find these information is to use online tools like [this](https://mxtoolbox.com/SuperTool.aspx?action=mx%3alessonsec.com&run=toolpage#), but I think that it's better to query the DNS server manually to better understand how these things work.

## Actually spoofing an email

Now that we have an overview of email security, we can go ahead and try to actually spoof an email.

Now, if you tried sending an email via code in past, you know for sure that it's really hard to avoid ending up in the spam folder, even if you are trying to send an email from a legitimate domain and you are authorized. Said that, let's go through some experiments I made and examine how feasible it really is to send spoofed emails from scratches.

Anyway, let's dive into it.

First of we need to setup a vulnerable server or try to find a vulnerable domain administrator that will allow you to spoof their domain name. Luckily using bug bounty programs it's easy to find vulnerable domains you're allowed to work with.

Now, if you are familiar on the "raw" email format, you'll know that the email is nothing but a text composed of headers-values and the content of the email. For this reason it's possible to "spoof" headers like `From`, `Reply-to`, `Sender`.

### First attempt: using Gmail SMTP server

I tried using my gmail account to send an email using google servers.
To do that I had to enable access from less secure apps on my account. This is the only way to login into a google service from code you write.

Now, it's not possible to specify envelope information, but it's possible to send the email and specify the `From:`, `Reply-to:` and `Sender:` headers. 
Using the `From:` field, I was able to change the sender name, but the email displayed by the client would be the one from the account used to login into gmail. If no `From:` header was specified, the header will be inferred by gmail servers and the email address (without the domain) will be used. Specifying the `Sender:` header didn't do much, if it was simply ignored, both with or without the `From:` header.
The `Reply-to:` header was interpreted just like Sender:, but if specified, emails would be marked as "promotions" by gmail's client.

Using different SMTP servers or clients will possibly lead to different results.

This is the code snippet used for this experiment:

```python
import smtplib

gmail_user = 'YOUR-EMAIL-HERE@gmail.com'
gmail_password = 'YOUR-PASSWORD-HERE'

sent_from_name = "Foo"
sent_from = 'foobar@spam.com'
to = 'receiver@email.com'
subject = 'This is a spoofed email'
body = "This is the content of a spoofed email"


email_text = f"""\
MIME-Version: 1.0
Date: Thu, 29 Apr 2021 18:49:13 +0200
Subject: {subject}
From: {sent_from_name} <{sent_from}>
To: {to}

{body}
"""

try:
    server = smtplib.SMTP_SSL('smtp.gmail.com', 465)
    server.ehlo()
    server.login(gmail_user, gmail_password)
    server.sendmail(sent_from, to, email_text)
    server.close()

    print("Email sent!")
except Exception as e:
    print(e)
    print("Email not sent!")
```

### Second attempt: setting up your personal SMTP server

Installing an SMTP server on your machine would make it possible to send emails from your computer using a script similar to the one used for google. Sadly this experiment won't let you send emails, this is for different reasons: for starters the IP address from which the email is sent is from a residential IP range. This means that if your ISP doesn't block the email (very possible), the receiving client will ignore the email because of the poor IP reputation.

I think it's possible to send emails using this technique if an AWS, GCP or Azure VM is used, this simply because the IP wouldn't be from a residential range anymore. I didn't try this yet, but I'm confident that emails would be at least delivered.

### Third attempt: using online services

With this approach I was able to send spoofed emails! I'm not linking any of the services I tried using, but there are free or paid services that actually work and let you specify the headers we are interested into.

Sadly most of the emails will end up in the spam folder or discarded by the client due to bad IP reputation (I think).
But it's a fun experiment to see how the emails are treated. Using a client with less protections (like online temporary email addresses ones) will allow the reception of the email.

## Conclusions

I think that the main take-away from this article should be an increased awareness about email security, mainly because of a deeper understanding of spoofing techniques.  For me personally, experimenting and researching on this topic was useful because I could work with DNS queries and gather insights about DNS security, a topic I'd love to further explore.

Sadly I couldn't experiment much at the moment, but I will for sure try and send emails from a VPS or Virtual Machine in the cloud that use a non-residential IP, I'm confident this approach would work in sending spoofed emails on vulnerable domains.  

--------------------------

This article was reviewed by Nicola Selenu from [topdeliverability.com](http://topdeliverability.com), big thanks to him!
