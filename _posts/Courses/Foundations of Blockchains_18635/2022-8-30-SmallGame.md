---
title: Foundations of Blockchains_18635 Note
date: 2022-08-30 22:30:00 -0400
categories: [Courses, Foundations of Blockchains_18635]
tags: [Blockchains]
img_path: /assets/img/Courses/Foundations of Blockchains_18635
---

An interesting card love dating game:

Target: Alice and Bob, if both love each other, dating; otherwise, not dating

<font color=Blue>Prerequisite:</font> honest-but-curious, can't be 'malicious'

Method 1:
❤️: love ♣️: not love

Step 1: Alice went into the room, puts card face-down on table, ❤️ if love, ♣️ if no love
Step 2: Alice leaves the room, Bob went into the room, if he loves Alice, open the card and get the result; otherwise, do nothing.
Step 3: Bob return to tell Alice if they would date.

It is like 'And' function, only 1 will make an affect on the result. So if the person inputs 1, he/she should watch the other's option; if the input is 0, then the result must be failure, so there is no need to watch the other's choice and thus protecting the privacy of the other one that may have ❤️.

Method 2:
![picture 1](Solution2.png)
_Solution2_

Secure 2-party computation

Alice and Bob want to learn some function on their joint inputs without disclosing their inputs, how to do it without cards? how to do it for any function?
