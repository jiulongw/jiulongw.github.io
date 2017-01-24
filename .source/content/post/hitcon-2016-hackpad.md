+++
title = "HITCON 2016 - Hackpad"
date = "2016-10-29T21:08:39-07:00"
tags = ["CTF", "Forensics", "Crypto"]
+++

In cryptography, a padding oracle attack is an attack which is performed using the padding of a cryptographic message.

<!--more-->

# Problem

> Hackpad  
> 65 Teams solved.
>
> Description  
> My site was hacked. The secret was leaked.  
> [hackpad.pcap.xz](https://s3-ap-northeast-1.amazonaws.com/hitcon2016qual/hackpad.pcap.xz_968494cea2c29140ee5e63e37c19cff2254f0229)
>
> Hint  
> None

# Solution

Load the pcap file in [Wireshark](https://www.wireshark.org/).  It contains series of HTTP requests.  Most
of the response codes are 500s with some 200s in between.  Below are some examples.

```
GET /
200 OK: encrypt(secret): 3ed2e01c1d1248125c67ac637384a22d997d9369c74c82abba4cc3b1bfc65f02...

POST /
msg=3ed2e01c1d1248125c67ac637384a22d997d9369c74c82abba4cc3b1bfc65f02...
200 OK: md5(decrypt(msg)) = e5d3583f3e05b9242a1933fd5d245200

POST /
msg=000000000000000000000000000000ff997d9369c74c82abba4cc3b1bfc65f02
500 Internal Server Error

POST /
msg=000000000000000000000000000000fe997d9369c74c82abba4cc3b1bfc65f02
500 Internal Server Error

POST /
msg=000000000000000000000000000000fd997d9369c74c82abba4cc3b1bfc65f02
500 Internal Server Error

...

POST /
msg=67acd06f7f7b28762310ce1213fccb11997d9369c74c82abba4cc3b1bfc65f02
200 OK: md5(decrypt(msg)) = d41d8cd98f00b204e9800998ecf8427e
```

It's pretty easy to guess what the server was doing:

1. It returns encrypted message on GET request
2. It decrypts the message in POST request and returns hash of plain message.
3. However, if the encrypted message in the POST request was malformed, it returns 500 Internal Server Error.

This is a typical [Padding Oracle Attack](https://en.wikipedia.org/wiki/Padding_oracle_attack).

In CBC mode decryption: P<sub>i</sub> = D(C<sub>i</sub>) &oplus; C<sub>i-1</sub> where C<sub>0</sub> = IV.

![CBC mode decryption](/img/hitcon-2016-hackpad/cbc_decryption.png)

We know `67acd06f7f7b28762310ce1213fccb11997d9369c74c82abba4cc3b1bfc65f02` is a valid encrypted message, it's
padding is `10101010101010101010101010101010`.  As a result:

D(`997d9369c74c82abba4cc3b1bfc65f02`) &oplus; `67acd06f7f7b28762310ce1213fccb11` = `10101010101010101010101010101010`

P<sub>1</sub> = D(`997d9369c74c82abba4cc3b1bfc65f02`) &oplus; `3ed2e01c1d1248125c67ac637384a22d`

Let's give it a quick try.

```
>>> a = 0x67acd06f7f7b28762310ce1213fccb11 ^ 0x10101010101010101010101010101010 ^ 0x3ed2e01c1d1248125c67ac637384a22d
>>> hex(a)[2:-1].decode('hex')
'In cryptography,'
```

Well, seems like we're on the right track.  The rest is simply cutting the original encrypted messages into 16 bytes blocks
and xor-ing it with the first block of every successful padding oracle attack result.

You can use `strings hackpad.pcap | grep msg=` to extract all messages and do some pre-processing to get a list of
succesful padding oracle attacks.

```
In cryptography,
 a padding oracl
e attack is an a
ttack which is p
erformed using t
he padding of a 
cryptographic me
ssage.
hitcon{H4
cked by a de1ici
0us pudding '3'}
```

Flag: `hitcon{H4cked by a de1ici0us pudding '3'}`

Credits to team member @jina.
