---
title: Nara - Intermediate - Windows - PG Practice
date: 2024-06-12
tags:
  - pg-practice
  - offsec
  - oscp-prep
  - ospg
  - windows
draft: false
---
# Nara - Intermediate - Windows - PG Practice

# **Scanning**

My basic scan runs the following command
```bash
nmap -sC -T5 -Pn -p- --min-rate=10000 $IP
```

```python
┌──(kali㉿kali)-[~/…/PGPractice/Easy/Windows/Nara]
└─$ basic 192.168.223.30         
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-12 05:24 EDT
Warning: 192.168.223.30 giving up on port because retransmission cap hit (2).
Nmap scan report for 192.168.223.30
Host is up (0.25s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=Nara.nara-security.com
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:Nara.nara-security.com
| Not valid before: 2023-07-30T14:09:26
|_Not valid after:  2024-07-29T14:09:26
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
| ssl-cert: Subject: commonName=Nara.nara-security.com
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:Nara.nara-security.com
| Not valid before: 2023-07-30T14:09:26
|_Not valid after:  2024-07-29T14:09:26
|_ssl-date: TLS randomness does not represent time
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
| ssl-cert: Subject: commonName=Nara.nara-security.com
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:Nara.nara-security.com
| Not valid before: 2023-07-30T14:09:26
|_Not valid after:  2024-07-29T14:09:26
|_ssl-date: TLS randomness does not represent time
3389/tcp  open  ms-wbt-server
| rdp-ntlm-info: 
|   Target_Name: NARASEC
|   NetBIOS_Domain_Name: NARASEC
|   NetBIOS_Computer_Name: NARA
|   DNS_Domain_Name: nara-security.com
|   DNS_Computer_Name: Nara.nara-security.com
|   DNS_Tree_Name: nara-security.com
|   Product_Version: 10.0.20348
|_  System_Time: 2024-06-12T09:25:57+00:00
| ssl-cert: Subject: commonName=Nara.nara-security.com
| Not valid before: 2024-05-06T12:45:53
|_Not valid after:  2024-11-05T12:45:53
|_ssl-date: 2024-06-12T09:25:57+00:00; 0s from scanner time.
5985/tcp  open  wsman
49664/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown
49678/tcp open  unknown
49682/tcp open  unknown
49710/tcp open  unknown

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-06-12T09:25:59
|_  start_date: N/A

Nmap done: 1 IP address (1 host up) scanned in 168.85 seconds
```

