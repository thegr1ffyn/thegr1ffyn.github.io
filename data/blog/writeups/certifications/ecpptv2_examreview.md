---
title: eCPPTv2 Exam Review (2024)
date: '2024-02-08'
tags: ['ecppt', 'ecpptv2', 'certification', 'exam-review', 'cert-review', 'ecppt-review']
draft: false
---
# eCPPTv2 Exam Review (2024)
I started my eCPPTv2 exam on February 2nd, 2024 and pwned all the 5 machines on 5th. Submitted my report the next day and received a much awaited email the same day stating:

`Hello Hamza, Congratulations! You are now an eCPPTv2! Your shiny certificate is waiting for you." `

## Background
Preparing for the eLearnSecurity Certified Professional Penetration Tester (eCPPTv2) exam is really important because it's a big deal in the world of cybersecurity. This certification, offered by INE/eLearnSecurity, is all about testing your skills in things like finding weaknesses in websites, and networks, and dealing with security issues.

The exam itself is hands-on and practical, which means you'll be doing real tasks to show what you can do. It's not just about answering questions – you'll actually be testing out your skills in a simulated environment.

One important part of the exam is writing up a professional report based on what you find. This is a key skill in the field because you need to be able to communicate your findings clearly. After you submit your report, it usually takes about a month to get your results back.

## Who Am I?
I am an undergraduate student currently starting my 6th semester (as of now) with a little over one year of experience in penetration testing. I am a regular CTF player for a year. Other than that, I am a Cyber Security Challenge Developer where I develop CTF challenges for Web, Reversing, Forensics, and Boot2Root.

## Why I chose eCPPT without eJPT?
I went through the eJPT course content and found it to be the same as my knowledge in November 2023 so I decided to take a leap of faith and prepare myself for a higher cert.

## Material and Labs
I bought just the exam voucher for 400 USD. Since I didn't have the official course material, I opted for self-study focused on the following topics.

- Pivoting (*must*)
- Port Forwarding
- Enumeration
- Buffer Overflow
- Privilege Escalation
- Proxychains
- Metasploit/Meterpreter   

## Preparation
Here are some useful resources that I found during my study which will give you a rough idea about what to do when preparing.

### Pivoting
This is a big portion of the exam. Practice using autoroute / proxy chains / port forwarding. Metasploit 6 might give you a tough time due to deprecated scripts and you will have to work your way around it.

- [Wreath](https://tryhackme.com/room/wreath) on THM
- [This](https://youtu.be/QNoIX1au_CM?si=0aU0FM5TzvB2QcCv) is a great video tbvh by [OvergrownCarrot1 Hacking](https://www.youtube.com/@overgrowncarrot1), helped me clear most of my concepts
- Practice the [Active Directory Path](https://tryhackme.com/module/hacking-active-directory) on TryHackMe
- This is a [great article](https://pentest.blog/explore-hidden-networks-with-double-pivoting/) and explains the double pivoting concepts very well.
- [Pivoting and Relaying Techniques using Meterpreter](https://medium.com/axon-technologies/how-to-implement-pivoting-and-relaying-techniques-using-meterpreter-b6f5ec666795)

### HackTheBox
Honestly, just practice Linux/Windows Priv Escalation. After spending some time doing HackTheBox to build a stronger foundation, if you are able to understand and replicate the steps for the Easy/Medium Boxes in HackTheBox, you're good to go.

### TryHackMe
Do practice the following rooms on THM. They will help you during your exam.
- Wreath 
- Internal
- Empline
- ContainMe

### Buffer Overflow
Being average at Buffer Overflows, this was the part I was most scared of but I found it to be the most interesting part of the exam. Follow this checklist when preparing for BOFs since they can get a little tricky.
- [TCM Buffer Overflow Video](https://www.youtube.com/playlist?list=PLLKT__MCUeix3O0DPbmuaRuR_4Hxo4m3G)
- [Tiberius BOF Room](https://tryhackme.com/room/bufferoverflowprep) on THM
- [Gatekeeper***](https://tryhackme.com/room/gatekeeper) on THM
- [Brainpan**](https://tryhackme.com/room/brainpan) on THM

### Exploit Development
- https://github.com/Tib3rius/Pentest-Cheatsheets/blob/master/exploits/buffer-overflows.rst
- https://github.com/TheFlash2k/bof-scripts

### Reporting
Reporting is a crucial element of the exam. Make sure to take ALOT of screenshots of each step as it will help you redo your exam in case you mess up with the machine and have to reset it. Although there is MORE than enough time for reporting which I think, to be honest should only be a day or two. 

I used [`OSCP Template`](https://github.com/noraj/OSCP-Exam-Report-Template-Markdown) by [Noraj](https://github.com/noraj) which takes your Markdown file and converts it to a properly formatted PDF file. I highly recommend using this tool. You can use Notion or Obsidian to make regular notes during your exam.

- https://www.youtube.com/watch?v=NEz4SfjjwvU&list=WL&index=12&ab_channel=ITPro
- https://www.youtube.com/watch?v=Ohm0LhFFwVA&ab_channel=Conda

### Additional Resources
- [eCPPT Field Manual](https://drive.google.com/file/d/1wC7RMTrWjt74rO8u4X-zM89T_hZzF_A5/edit): probably a blessing as it this genuinely will help you alot.
- [Pivoting Cheatsheet](https://www.sans.org/posters/pivot-cheat-sheet/)
- [Red Team Notes](https://github.com/thegr1ffyn/Notes/blob/main/Red%20Team%20Notes.md)

Overall, it was a fun exam. Don't OVERCOMPLICATE things, take multiple breaks, and enjoy the exams. The VPN connection used OpenVPN and it was very smooth and stable. If you know how to pivot properly and work your way around Windows/Linux boot2root machines then you should have little to no problem doing this exam.

Lastly some **TIPS:**
- **Screenshot EVERYTHING**. I mean, everything. Your commands, web pages, scans, any exploit code, etc. It will save you a LOT of time during the report.

- **ENUMERATE EVERYTHING**. Check everywhere. Don't just depend on enumeration scripts, poke around at everything.

- Make sure you take good notes and use a program that you can add screenshots to easily. I used Notion and it was perfect.

- Read over your report like 4-5 times. Make sure you include everything and plus point if you look into the details like formatting, overall look, and presentation, and add diagrams where needed.

- For pivoting, whenever you feel stuck, draw your network and debug how networks talk to each other.

- Remember to sleep and take breaks. Don't be like me who rooted 4/5 machines in less than 8 hours and got brain drain for the last pivot. Give it your time and enjoy the exam. 

So this was it, gonna start studying for **OSCP** now xD

Last but not least, here is my shiny certificate ;)
![cert](/static/writeups/certifications/ecppt.png)

