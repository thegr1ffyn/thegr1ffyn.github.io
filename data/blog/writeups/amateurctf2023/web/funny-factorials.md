---
title: AmateurCTF '23 - Web - Funny Factorials
date: '2023-07-18'
tags: ['ctf', 'web', 'amateurctf', 'amateurctf23', 'writeup', 'lfi']
draft: false
summary: Utilizing LFI in the theme parameter to get the flag.
---

## Challenge Description

I made a factorials app! It's so fancy and shmancy. However factorials don't seem to properly compute at big numbers! Can you help me fix it?

Author: stuxf

Connection info: [funny-factorials.amt.rs](https://funny-factorials.amt.rs)

## Solution

Firstly, we were provided with two files. `app.py` and a `Dockerfile`

```docker:Dockerfile
FROM python:3.10-slim-buster

RUN pip3 install flask
COPY flag.txt /

WORKDIR /app
COPY app/* /app/
copy app/templates/* /app/templates/
copy app/themes/* /app/themes/

EXPOSE 5000

ENTRYPOINT ["python3", "app.py"]
```

Seeing this, we can that the flag is being stored in `/`. Let's firstly analyze the `app.py` file and try to find a bug using code analysis

```python:app.py
from flask import Flask, render_template, request
import sys

app = Flask(__name__)

def factorial(n):
    if n == 0:
        return 1
    else:
        try:
            return n * factorial(n - 1)
        except RecursionError:
            return 1

def filter_path(path):
    # print(path)
    path = path.replace("../", "")
    try:
        return filter_path(path)
    except RecursionError:
        # remove root / from path if it exists
        if path[0] == "/":
            path = path[1:]
        print(path)
        return path

@app.route('/')
def index():
    safe_theme = filter_path(request.args.get("theme", "themes/theme1.css"))
    f = open(safe_theme, "r")
    theme = f.read()
    f.close()
    return render_template('index.html', css=theme)

@app.route('/', methods=['POST'])
def calculate_factorial():
    safe_theme = filter_path(request.args.get("theme", "themes/theme1.css"))

    f = open(safe_theme, "r")
    theme = f.read()
    f.close()
    try:
        num = int(request.form['number'])
        if num < 0:
            error = "Invalid input: Please enter a non-negative integer."
            return render_template('index.html', error=error, css=theme)
        result = factorial(num)
        return render_template('index.html', result=result, css=theme)
    except ValueError:
        error = "Invalid input: Please enter a non-negative integer."
        return render_template('index.html', error=error, css=theme)

if __name__ == '__main__':
    sys.setrecursionlimit(100)
    app.run(host='0.0.0.0')
```

By simply analyze the provided code, we understood the following points

- GET request to `/` along with `theme` parameter allows LFI.
- The `filter_path` function can be bypassed using two different methods.
    - Using `....//` in the payload. This is because `../` will be replaced with an empty string and we'll get the `../` back. BUT, the problem is, there is `filter_path(path)` function call. Which will keep on calling the function until we don't have ../ in the path. Which means, in order to bypass, we need to use `//`. This will bypass the `if path[0] == "/": path = path[1:]` condition as the first `/` will be removed and we'll still have `/`
- The flag exists in `/` directory

### Exploitation

Well, the payload is pretty simple, we can try and get the flag using the following two payloads:

[https://funny-factorials.amt.rs/?theme=//flag.txt](https://funny-factorials.amt.rs/?theme=//flag.txt)

Now, the problem is, we don't see the flag on the page:

![chal](/static/writeups/amateurctf2023/web/ff-1.png)

That is because, the theme is loaded into the `<style>` html tags and those aren't rendered directly. So view that, we'll see the source code

![chal](/static/writeups/amateurctf2023/web/ff-2.png)

And we get the flag! Also, we can use the following bash one-liner to get the flag

```bash
curl -sL 'https://funny-factorials.amt.rs/?theme=//flag.txt' | grep ama | tr -d ' '
```

![chal](/static/writeups/amateurctf2023/web/ff-0.png)
