---
title: Ignite Hackathon '22 - Forensics - PSEncrypt
date: '2023-01-05'
tags: ['ctf', 'forensics', 'ignite', 'ignite22', 'writeup', 'network-forensics', 'wireshark', 'powershell']
draft: false
summary: Can you get the flag from this weird file?
---
# PSEncrypt 

This question was given in the Grande Finale of Digital Pakistan Cybersecurity Hackathon 2022. It belonged to Network Security Category and had 100 points.

We were given a pcap file named file.pcapng. Lets open it in Wireshark.


![Untitled](static/writeups/ignite22/forensics/Untitled.png)

We are given 3508 packets to analyze. Lets filter it out using by

File >> Export Objects >> HTTP

![Untitled](static/writeups/ignite22/forensics/Untitled%201.png)

We get a pop-up with the filtered HTTP object list. We save them all in a folder. Our Wireshark task is completed.

![Untitled](static/writeups/ignite22/forensics/Untitled%202.png)

Here we are given a host.ps1 (PowerShell file) contains the encrypt and decrypt functions and other parameters.

We are missing one element in the Decrypt-String which we found in the razor(48) file saved in the same folder. It is a base64 string” IN3DZMA9y5D0q5y4Pe3Uv%2FVE3mA4EZY55XHJJIdLc29WAK73bE2DzB7ae%2Fmpy4CW”

Install powershell on linux using command

`apt install powershell`

Open the file host.ps1 and delete the last two call to functions and replace it with 

`Decrypt-String $key "IN3DZMA9y5D0q5y4Pe3Uv/VE3mA4EZY55XHJJIdLc29WAK73bE2DzB7ae/mpy4CW”`

Save the file and execute on linux.

`pwsh host.ps1`

We have the flag 😎

![Untitled](static/writeups/ignite22/forensics/Untitled%203.png)

`flag: flag{**_****_Chi11}`