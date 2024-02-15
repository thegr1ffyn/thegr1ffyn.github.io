---
title:  why - Forensics - Cyber Siege 23
date: '2023-03-25'
tags: ['why', 'forensics', 'cyber-siege', 'ctf', 'capture-the-flag']
draft: false
---

# why - Forensics - Cyber Siege 23

## Hint: I don't know why.

## Solution:

Given is a file named why.rar which is password protected. The password is pretty simple and hidden in plain sight. Yes, you guessed it right, “why”. 

![image](https://user-images.githubusercontent.com/95119705/221403367-c109ad0f-8da1-4b1b-a5c2-5217304360d0.png)


We get another zip file that is not password protected. Thank God!

We have two files in the folder.

![image](https://user-images.githubusercontent.com/95119705/221403376-bdb0910e-77fd-47b7-abc9-c9b49fa465df.png)

The audio is some total gibberish. while the file flag.txt is a ```.exe``` which is locked. 

Audacity or Sonic Visualizer is our best friend. We open the audio.

![image](https://user-images.githubusercontent.com/95119705/221403389-35e5b656-1690-4a39-ada4-2937c7ff55e4.png)

The first thing to always check for in an audio file during any CTF is to check its spectrogram for any information. Let's see:

![image](https://user-images.githubusercontent.com/95119705/221403403-6baf150b-247f-406f-b560-7eea5b1738bc.png)

![image](https://user-images.githubusercontent.com/95119705/221403409-89b89b50-1cd6-44ad-b529-d8114832a9b5.png)

and VOILAAAAA!! We have a string that seems like Base64 encoded. Let's fire up Cyberchef and decrypt it.

![image](https://user-images.githubusercontent.com/95119705/221403417-daf57ee2-a581-442d-a4b8-09393fd0012b.png)

Seems like the question is giving us a tough time by telling us that we are on a dead end and the string is a password, not a flag. But wait, we have another file given to us that is password protected. This might be its password. Let's check:

![image](https://user-images.githubusercontent.com/95119705/221403431-686661ee-6c89-4638-a8db-242ff28ff8a7.png)

```iamapasswordnotaflag``` is the password to the encrypted file. Now we have a file named flag.txt. Open it and you will find your flag.

```AOF{aud10_f0ren$1c$_1$_fun}```
