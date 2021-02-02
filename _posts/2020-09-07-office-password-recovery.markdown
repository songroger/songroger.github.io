---
layout: post
title:  "docx cracked experience"
date:   2020-09-07 20:20:39 +0000
categories: code
---

1. Hash file

Use `john`[^footnote1] to get the hash file.

`python john.py 2020.docx >hash.txt`

2. Hashcat[^footnote2] to crack the password

Example commands for hashcat:

`hashcat -a 3 -m 9400 --username -o cracked_pass.txt 111.txt song?d?d?d --force --self-test-disable`

`hashcat -a 0 -m 9400 rockyou.txt 111.txt --username -o cracked_pass.txt --force --self-test-disable`

`hashcat -a 3 -m 9600 hash.txt test.hcmask --username -o cracked_pass.txt --force --self-test-disable`


[^footnote1]:
[john](/img/john.py)

[^footnote2]:
[hashcat](https://github.com/hashcat/hashcat)