---
title: 漏洞利用学习笔记-010-HeapSpray
description: Heap Spray - 堆与栈的协同攻击
date: 2020-10-25 17:56:00+0800
categories:
    - 漏洞利用
tags:
    - Windows
    - 堆溢出
    - 栈溢出
---

声明：实验环境为 Windows 2000

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞利用学习笔记-010-heapspray/)

## 简介

> 针对浏览器的攻击中，常常会结合使用堆和栈协同利用漏洞

- 当浏览器或其使用的ActiceX控件中存在溢出漏洞时，攻击者就可以生成一个特殊的HTML文件来触发这个漏洞
- 不管是堆溢出还是栈溢出，漏洞触发后最终能够获得EIP
- 有时我们可能很难在浏览器中复杂的内存环境下不知完整的shellcode
- 页面中的JavaScript可以申请堆内存，因此shellcode通过JavaScript布置在堆中称为可能
- shellcode放在堆中如何定位：HeapSpray

## 技术细节

- 在使用Heap Spray的时候，一般将EIP指向堆区`0x0C0C0C0C`位置，然后用JavaScript申请使用大量堆内存并用包含着`0x9`0和shellcode的内存片覆盖这些内存
- 通常，JS会从低地址向高地址分配内存，因此申请的内存超过200MB的话，0x0C0C0C0C将被含有shellcode的内存片覆盖，只要内存片中的0x90能够命中0x0C0C0C0C，shellcode就可以执行

## JS代码

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

- 解释
  - 每个内存片大小为1MB
  - 首先产生一个大小为1MB而且全部被0x90填满的内存块
  - 由于Java会为申请到的内存填上一些额外的信息，为了保证内存片是1MB，要将这些空间减去
  - 我们使用200个这样的内存片来覆盖堆内存，只要任意一篇nop区可以覆盖0x0C0C0C0C，就可以成功

> 额外空间

|  | size | 说明 |
| - | - | - |
| malloc header | 32 byte | 堆块信息 |
| string length | 4 byte | 表示字符串长度 |
| terminator 2 | 1 byte | 堆块信息 |

## 实践

- 未完待续
