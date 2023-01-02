---
title: Exploit Learning Notes 003 API Dynamic Loading
description: ShellCode's API Dynamic Loading
date: 2020-10-19 20:20:00+0800
categories:
    - Exploit
tags:
    - Windows
    - ShellCode
---

> [Click here to download this article with executable program, shellcode file](exploit-study-03.zip)

Source: [Moeomu's blog](/posts/exploit-learning-notes-003-api-dynamic-loading/)

## Locate the API address via TEB

### Locate Kernel32.dll

- When the program is loaded, the `[FS:0]` register in the user state holds the TEB address
- TEB offset 0x30 at location `[TEB + 0x30]` holds the PEB address
- PEB offset 0xC location `[PEB + 0xC]` holds PEB_LDR_DATA
- The official Microsoft description of the PEB_LDR_DATA structure [click here](https://docs.microsoft.com/en-us/windows/win32/api/winternl/ns-winternl-peb_ldr_data) is represented in C as follows

```CPP
typedef struct _PEB_LDR_DATA {
  BYTE       Reserved1[8];
  PVOID      Reserved2[3];
  LIST_ENTRY InMemoryOrderModuleList;
} PEB_LDR_DATA, *PPEB_LDR_DATA;
```

> Here are the results of my debugging

```cpp
typedef struct _PEB_LDR_DATA {
  INT        Length;
  UCHAR      Initialized;
  PVOID      SsHandle;
  LIST_ENTRY InLoadOrderModuleList;
  LIST_ENTRY InMemoryOrderModuleList;
  LIST_ENTRY InInitializationOrderModuleList;
  PVOID      EntryInProgress;
  UCHAR      ShutdownInProgress;
  PVOID      ShutdownThreadId;
} PEB_LDR_DATA, *PPEB_LDR_DATA;
```

- We need to read the InInitializationOrderModuleList to get the address of Kernel32.dll, and this list is the LIST_ENTRY structure, the official Microsoft description of this structure is as follows

```CPP
typedef struct _LIST_ENTRY {
   struct _LIST_ENTRY *Flink;
   struct _LIST_ENTRY *Blink;
} LIST_ENTRY, *PLIST_ENTRY, *RESTRICTED_POINTER PRLIST_ENTRY;
```

> Here are the results of my debugging

```CPP
typedef struct LinkNode
{
    _LIST_ENTRY Flink;
    _LIST_ENTRY Blink;
    PVOID DllAddress;
}
```

- This shows that DllAddress exists at `+0xC` of the chain table node, while the first three nodes of any program are `Ntdll -> KernelBa -> Kernel32`

### Locate the API address (reverse Kernel32.dll)

> The previous section obtained the load base address of Kernel32, from which this section obtains the addresses of `LoadLibrary` and `GetProcAddress` for other functions

- The offset `0x3C` is the entry point of PEHeader, the flag word is `0x5045` and the text is `PE`.
- plus the offset of `0x78` is the address of the Export Directory RVA, at this time the offset is `0x168` and the value is `0x262C`
- plus the offset of `0x4` is the Export Directory Size, which is `0x6CFD`.
- When on disk, the minimum unit of section size is `0x200`, but when loaded into memory, the minimum unit of section size becomes `0x1000`, while the PE file header occupies a size of `0x400` in the file, but will occupy a size of `0x1000` when mapped into memory, the size difference is `0xC00`, so `0x262C` should be subtracted from `0xC00` to get the address `0x1A2C` of the export directory table `Export Directory`.
- In the export directory table `0x28` offset is the address of the first export function, the sequence number is `0`
- At offset `0x67C` in the Export Directory table is the address of the function `GetProcAddress` with the serial number `198`.
- At the `0x340` offset in the export directory table is the address of the function `LoadLibraryA`, with the serial number `244`.
- At this point, the addresses of the two important functions are found

---

## Debug ShellCode(ExpStd0301)

```C
char shellcode[] = "\x10\x10";

void main()
{
  __asm
  {
    lea eax, shellcode
    push eax
    ret
  }
}
```

---

## Dynamic API loading ShellCode

### Theoretical analysis

- Required API functions
  - MessageBoxA(User32.dll)
  - ExitProcess(Kernel32.dll)
  - LoadLibraryA(Kernel32.dll)
- A puzzle: how to find the address of the API when the ShellCode is needed to be as short as possible (no function name exists)

### HASH algorithm for function names

#### Theory

- Need to introduce an additional HASH algorithm
- The result of the calculation is called DIGEST (summary)
- HASH of the searched function name

#### algorithm (ExpStd0302)

```C
#include <stdio.h>
#include <windows.h>

DWORD GetHash(char *fun_name)
{
  DWORD digest = 0;
  while(*fun_name)
  {
    digest = ((digest << 25) | (digest >> 7));
    digest += *fun_name;
    fun_name++;
  }
  return digest;
}

void main()
{
  DWORD hash;
  hash = GetHash("MessageBoxA");
  printf("Hash is %s", hash);
}
```

---

## Final ShellCode(ExpStd0303)

```x86asm
int main()
{
  _asm{
    ;flag
    nop
    nop
    nop
    nop
    nop

    cld                        ;clear flag DF

    ;store hash
    push 0x1e380a6a            ;hash of MessageBoxA
    push 0x4fd18963            ;hash of ExitProcess
    push 0x0c917432            ;hash of LoadLibraryA
    mov esi, esp               ;esi = addr of first func hash
    lea edi, [esi-0xc]         ;edi = addr to start writing func

    ;make some stack space
    xor ebx, ebx
    mov bh, 0x04
    sub esp, ebx

    ;push a pointer to "user32" onto stack
    mov bx, 0x3233             ;rest of ebx is null
    push ebx
    push 0x72657375
    push esp
    xor edx, edx

    ;find base addr of kernel32.dll
    mov ebx, fs:[edx + 0x30]   ;ebx = PEB address
    mov ecx, [ebx + 0x1c]      ;ecx = loader data pointer
    mov ecx, [ecx + 0x1c]      ;ecx = first entry in Initialization order list
    mov ecx, [ecx]             ;ecx = second entry
    mov ebp, [ecx + 0x08]      ;ebp = base address of kernel32.dll

  find_lib_functions:
    lodsd                      ;load next hash into al and increment esi
    cmp eax, 0x1e380a6a        ;hash of MessageBoxA - trigger and LoadLibrary("user32")
    jne find_functions
    xchg eax, ebp              ;save current hash
    call [edi - 0x8]           ;LoadLibraryA
    xchg eax, ebp              ;restore current hash and update ebp with base address of user32.dll
  
  find_functions:
    pushad                     ;preserve registers
    mov eax, [ebp + 0x3c]      ;eax = start of PEheader
    mov ecx, [ebp + eax + 0x78];ecx = relative offset of export table
    add ecx, ebp               ;ecx = absolute addr of export table
    mov ebx, [ecx + 0x20]      ;ebx = relative offset of names
    add ebx, ebp               ;ebx = absolute addr of names table
    xor edi, edi               ;edi will count through the functions
  
  next_function_loop:
    inc edi                    ;inc function counter
    mov esi, [ebp + edi * 4]   ;esi = relative offset of current function name
    add esi, ebp               ;esi = absolute addr of current function name
    cdq                        ;dl will hold hash (we know eax is small)
  
  hash_loop:
    movsx eax, byte ptr[esi]
    cmp al, ah
    jz compare_hash
    ror edx, 7
    add edx, eax
    inc esi
    jmp hash_loop

  compare_hash:
    cmp edx, [esp + 0x1c]      ;compare to the requested hash(saved on stack from pushad)
    jnz next_function_loop
    mov ebx, [ecx + 0x24]      ;ebx = relative offset of ordinals table
    add ebx, ebp               ;ebx = absolute addr of ordinals table
    mov di, [ebx + 2 * edi]    ;di = ordinal number of matched function
    mov ebx, [ecx + 0x1c]      ;ebx = relative offset of address table
    mov ebx, ebp               ;ebx = absolute addr of address table
    add ebp, [ebx + 4 * edi]  ;add to ebp(base addr of module) the relative offset of matched function
    xchg eax, ebp              ;move func addr into eax
    pop edi                    ;edi is last onto stack in pushad
    stosd                      ;write function addr to [edi] and increment edi
    push edi
    popad                      ;restore registers and loop until we reach end of alst hash
    cmp eax, 0x1e380a6a
    jne find_lib_functions
  
  function_call:
    xor ebx, ebx
    push ebx                   ;cut string
    push 0x74736577
    push 0x6c696166            ;push failwest
    mov eax, esp               ;load address of failwest
    push ebx
    push eax
    push eax
    push ebx
    call [edi - 0x4]           ;call MessageBoxA
    push ebx
    call[edi - 0x8]            ;call ExitProcess


    ;flag
    nop
    nop
    nop
    nop
  }
}
```
