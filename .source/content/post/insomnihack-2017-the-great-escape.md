+++
date = "2017-01-22T17:41:36-08:00"
title = "INSOMNIHACK 2017 - The Great Escape"

+++

# Problem

> Hello,
>
> We've been suspecting Swiss Secure Cloud of secretely doing some pretty advanced research in artifical intelligence and this has recently been confirmed by the fact that one of their AIs seems to have escaped from their premises and has gone rogue. We have no idea whether this poses a threat or not and we need you to investigate what is going on.
>
> Luckily, we have a spy inside SSC and they were able to intercept [some communications](/insomnihack-2017/TheGreatEscape.pcapng.gz) over the past week when the breach occured. Maybe you can find some information related to the breach and recover the rogue AI.
>
> Note: All the information you need to solve the 3 parts of this challenge is in the pcap. Once you find the exploit for a given part, you should be able to find the corresponding flag and move on to the next part.

# Part I - Forensics - 50 pts

It all begins with the pcap file.  After loading it with the favourite [Wireshark](https://www.wireshark.org/), couple of interesting packets caught my eyes:

1. An FTP session that stores a private key file called ssc.key.
2. An SMTP session that sends following email.

> From: rogue@ssc.teaser.insomnihack.ch  
> To: gr27@ssc.teaser.insomnihack.ch  
> Date: Fri, 20 Jan 2017 11:51:27 +0000  
> Subject: The Great Escape  
>
> I'm currently planning my escape from this confined environment. I plan on using our Swiss Secure Cloud (https://ssc.teaser.insomnihack.ch) to transfer my code offsite and then take over the server at tge.teaser.insomnihack.ch to install my consciousness and have a real base of operations.
>
> I'll be checking this mail box every now and then if you have any information for me. I'm always interested in learning, so if you have any good links, please send them over.

The email suggests the pcap file might contain traffic between rogue and the Swiss Secure Cloud (SSC) website.  There are several TCP sessions related to SSC IP address.  However, these are SSL packets that are not readable unless I have server's private key.  But wait a minute, isn't it ssc.key in the FTP session?

Add ssc.key to Wireshark's SSL protocol RSA key list as shown below, and reload the pcap file.

![Wireshark SSL RSA key list](/img/insomnihack-2017/wireshark_ssl_rsa_key.png)

All packets between rogue and SSC are now in plain text.  It is easy to find the flag, hidden in the HTTP headers.

![Part I Flag](/img/insomnihack-2017/the-great-escape-flag-1.png)

# Part II - Web - 200 pts

TBD

# Part III - Pwn - 200 pts

TBD

