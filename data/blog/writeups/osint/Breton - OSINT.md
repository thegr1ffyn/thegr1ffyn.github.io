---
title: Breton | OSINT | Cyber Siege 2023
date: '2023-03-25'
tags: ['breton', 'osint', 'cyber-siege', 'ctf', 'capture-the-flag']
draft: false
---

# Breton | OSINT | Cyber Siege 2023
## Hint: What time and location did I last ping on last year’s Valentine's Day? Fun Fact: Last time I checked I had 1437 pounds in my account and I was tagged in 2020. Flag Format is AOF{xx_xx_xxAM_XXXXXXXXX} 

## Solution:

So we are being asked to find the location and time of ping of something. We are also told that it has 1437 pounds. The other hint tells us that it is under the water, so it makes sense that our subject is a sea creature with 1437 pounds weight. Lets google:

![image](https://user-images.githubusercontent.com/95119705/221402913-875d195f-042b-4485-86dc-c272fff64490.png)

The first article shows some motivation. Lets see:

![image](https://user-images.githubusercontent.com/95119705/221402916-b81cd1d5-638d-407d-ab28-f434df3bec7d.png)

So now we have found the name of the shark, Breton. Interestingly, the name of the challenge is also the name of the shark we are talking about.

Now, the challenge asks us to find the location and time when the shark last pinged on Valentine's Day. Now we know that Valentine's Day is on 14 February every year. We are told that we are finding last year's ping. this means that our date becomes 14 February 2022.

Lets research what website tracks shark activities. 

![image](https://user-images.githubusercontent.com/95119705/221402924-535ee788-b76a-417a-b2cf-440e3892d078.png)
Let's find location of Breton shark.

[OCEARCH Shark Tracker](https://www.ocearch.org/tracker/detail/breton)

![image](https://user-images.githubusercontent.com/95119705/221402940-b941502b-22dd-4111-827f-8978d0a356ce.png)

Here is our first half of flag.

AOF{7:49:46AM

Now to find second half, we can see on our screen that the location is **Onslow Bay**

This is our second half.

So the flag becomes:

AOF{7:49:46AM_ONSLOWBAY}
