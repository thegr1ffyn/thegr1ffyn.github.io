---
title: Hutch - Intermediate - Windows- PG Practice
date: 2024-06-12
tags:
  - pg-practice
  - offsec
  - oscp-prep
  - linux
  - ospg
  - active-directory
draft: false
---
# **Scanning**

My basic scan runs the following command
```bash
nmap -sC -T5 -Pn -p- --min-rate=10000 $IP
```

```python
└─$ basic 192.168.227.122
Starting Nmap 7.93 ( https://nmap.org ) at 2024-07-11 09:53 EDT
Nmap scan report for 192.168.227.122
Host is up (0.14s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
|_http-title: IIS Windows Server
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND DELETE MOVE PROPPATCH MKCOL LOCK UNLOCK PUT
| http-webdav-scan: 
|   WebDAV type: Unknown
|   Public Options: OPTIONS, TRACE, GET, HEAD, POST, PROPFIND, PROPPATCH, MKCOL, PUT, DELETE, COPY, MOVE, LOCK, UNLOCK
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, POST, COPY, PROPFIND, DELETE, MOVE, PROPPATCH, MKCOL, LOCK, UNLOCK
|   Server Date: Thu, 11 Jul 2024 13:54:22 GMT
|_  Server Type: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-07-11T13:53:50
|_  start_date: N/A

Nmap done: 1 IP address (1 host up) scanned in 41.86 seconds
```

Further enumerating `ldap` using the following command.
```python
nmap -n -sV -Pn --script "ldap* and not brute" $IP
```
We find that it is running a subdomain named `hutch.offsec` under *ldap*
```python
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: hutch.offsec, Site: Default-First-Site-Name)
| ldap-rootdse: 
| LDAP Results
```

Now performing a `ldapsearch` using the following command
```python
ldapsearch -v -x -b "DC=hutch,DC=offsec" -H "ldap://$IP$" "(objectclass=*)"
```

This gives us a username and a password
```bash
# Freddy McSorley, Users, hutch.offsec
dn: CN=Freddy McSorley,CN=Users,DC=hutch,DC=offsec
objectClass: top
sAMAccountName: fmcsorley
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Freddy McSorley
description: Password set to CrabSharkJellyfish192 at user's request. Please change on next login.
```

Username: `fmcsorley`
Password: `CrabSharkJellyfish192`

192.168.205.122
```
ldapsearch -x -H ldap://192.168.205.122 -D 'hutch.offsec\fmcsorley' -w 'CrabSharkJellyfish192' -b "DC=hutch,DC=offsec"
```

# **Exploitation**

```
{9BoE[[Tl(58Fd
```

# **Privilege Escalation**