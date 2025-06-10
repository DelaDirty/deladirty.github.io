---
title: "Shellcode-Encryption – for C# Paylods (OSEP)"
date: 2025-06-07                   
categories:
  - blog                          
tags:
  - AD
  - tooling
  - av-evasion
layout: single
excerpt: "Python scripts that will auto-generate keys and encrypt your shellcode for the OSEP exam."
header:
  teaser: /assets/images/shellcode.png
---


Python scripts that will auto-generate keys and encrypt your shellcode into a C# byte array for the OSEP exam.
<!--more-->

# Why was this created?
<small>I've seen a few students create this, but in C#. I wanted to create something that would work quickly and efficiently.
Every time you run the Python script, it re-encrypts your shellcode and returns the new keys along with your shellcode. As many of us know, OSEP is about evading AV as much as possible. Instead of using the same keys, why not have something that automatically generates them and makes it easier for us to continue working through the labs and exams?</small>

## How does it work?
<small>Both AES256 and XOR Python scripts work the same way. The AES-256 will generate a randomized AES key and IV and output your encrypted shellcode. The XOR will generate a randomized XOR key for your encrypted shellcode.</small>

<small>For usage tips check out my github readme files for both.</small> 


<a class="btn btn--primary" href="https://github.com/DelaDirty/Shellcode-Encryption" target="_blank" rel="noopener">View on GitHub</a>
