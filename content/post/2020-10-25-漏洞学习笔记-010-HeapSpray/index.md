---
title: Exploit learning notes 010 HeapSpray
description: Heap Spray - Heap and Stack Cooperative Attack
date: 2020-10-25 17:56:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

Disclaimer: The experimental environment is Windows 2000

Source: [Moeomu's blog](/posts/exploit-learning-notes-010-heapspray/)

## Introduction

> Attacks against browsers often use a combination of heap and stack co-option vulnerabilities

- When there is an overflow vulnerability in the browser or in the ActiceX control it uses, an attacker can generate a special HTML file to trigger the vulnerability
- Whether it is a heap overflow or a stack overflow, the vulnerability can eventually gain an EIP when triggered
- Sometimes it can be difficult to know the full shellcode in the complex memory environment of the browser
- The JavaScript in the page can request heap memory, so the shellcode is laid out in the heap via JavaScript as a possibility
- How to locate the shellcode in the heap: HeapSpray

## Technical details

- When using Heap Spray, the EIP is usually pointed to the heap area `0x0C0C0C0C` location, and then JavaScript is used to request the use of a large amount of heap memory and overwrite it with a memory slice containing `0x9`0 and shellcode
- Normally, JS allocates memory from low addresses to high addresses, so if you request more than 200MB of memory, 0x0C0C0C0C will be overwritten by the memory slice containing the shellcode, and as long as 0x90 in the memory slice can hit 0x0C0C0C0C, the shellcode can be executed

## JS code

```JavaScript
var nop = unescape("%u9090%u9090");

while(nop.length <= 0x100000/2)
{
    nop+=nop;
}//生成一个 1MB 大小充满 0x90 的数据块

nop = nop.substring(0, 0x100000/2 - 32/2 - 4/2 - shellcode.length - 2/2);
var slide = new Arrary();
for (var i = 0; i < 200; i++)
{
    slide[i] = nop + shellcode
}
```

- Explanation
  - Each memory slice is 1MB in size
  - First generate a memory block of size 1MB and all filled with 0x90
  - Since Java fills the requested memory with some extra information, to ensure that the memory slice is 1MB, this space is subtracted
  - We use 200 of these memory slices to cover the heap memory, as long as any nop area can cover 0x0C0C0C0C, it will work

> extra space

| size | description |
| - | - | - |
| malloc header | 32 byte | heap block information |
| string length | 4 byte | indicates the length of the string |
| terminator 2 | 1 byte | heap block information |

## Practice

- To be continued
