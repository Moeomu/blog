---
title: 漏洞学习笔记-011-其它类型的漏洞和Windows安全机制
description: 其它类型的漏洞和Windows安全机制概述
slug: exploit-011
date: 2020-10-25 18:50:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

本文来源：[Moeomu的博客](/p/exploit-011/)

## 格式化串漏洞

### printf中的缺陷

> 例子

```CPP
#include "stdio.h"

void main()
{
    int a = 44,b = 77;
    printf("a=%d, b=%d\n",a,b);
    printf("a=%d, b=%d\n");
}
```

- 上述代码中第二个调用缺少了输出数据的变量列表
- 然而第二次调用没有引起编译错误，程序正常执行

### 用printf读取内存数据

> 例子

```CPP
#include "stdio.h"

int main(int argc, char ** argv)
{
    printf(argv[1]);
}
```

- 当我们向程序传入普通字符串，得到普通字符串
- 但是如果带有格式控制符，可以读出栈中的数据

### 用printf向内存中写数据

> 例子

```CPP
#include "stdio.h"
int main(int argc, char ** argv)
{
    int len_print = 0;
    printf("before write: length=%d\n", len_print);
    printf("Misaka:%d%n\n",len_print, &len_print);
    printf("after write: length=%d\n", len_print);
}
```

- %n控制符计算出了输出的字符串的长度，然后将它写回了len_print变量中

## SQL注入攻击

### 原理

- 它源于PHP，ASP等脚本语言堆用户输入数据和解析时的缺陷

### 它不是二进制漏洞，在此不再讨论

## Windows安全机制

### 图灵机的缺陷

- 代码和数据没有明确区分，所以总存在一些问题
- 例如堆栈溢出攻击，加壳脱壳技术，变形病毒技术
- 跨站脚本攻击，SQL注入攻击同样都是利用此缺陷造成的

### Windows的变革

#### 宏观变革

- 增加了Windows安全中心
- 为Windows添加了防火墙
- 未经允许，Web弹窗和ActiveX控件安装将禁止
- IE7添加了筛选仿冒网站功能
- 添加了UAC(User Account Control)用户账户控制机制，防止恶意软件在未经许可的情况下在计算机上进行安装或者堆计算机进行更改
- 集成了Windows Defender，可以阻止，控制，删除间谍软件和恶意软件

#### 内存安全的变革

- 使用GS编译技术，在函数返回地址前加入了`SecurityCookie`，在函数返回前首先检测`SecurityCookie`是否覆盖，栈溢出变得困难
- 增加了堆SEH的安全校验机制，有效防止大多数改写SEH而劫持进程的攻击
- 堆中加入了`Heap Cookie`，`Safe Unlinking`等安全机制，堆溢出的限制更多
- `DEP(Data Execution Protection)`数据执行保护将数据部分标识为不可执行，阻止栈，堆和数据节中攻击代码的执行
- `ASLR(Address Space Layout Randomization)`加载地址随机化技术通过堆系统关键地址的随机化，使得经典的堆栈溢出手段失效
- `SEHOP(Structured Exception Handler Overwrite Protection)`SEH覆盖保护作为堆SEH安全机制的补充，将SEH的保护提升到系统级别，使得SEH的保护机制更有效

#### Windows安全机制汇总

|  | Windows XP | Windows 2003 | Windows Vista | Windows 2008 | Windows 7 |
| - | - | - | - | - | - |
| **GS** |   |   |   |   |   |
| 安全Cookies | √ | √ | √ | √ | √ |
| 变量重排 | √ | √ | √ | √ | √ |
| **安全SEH** |   |   |   |   |   |
| SEH句柄验证 | √ | √ | √ | √ | √ |
| **堆保护** |   |   |   |   |   |
| 安全拆卸 | √ | √ | √ | √ | √ |
| 安全快表 | × | × | √ | √ | √ |
| Heap Cookie | √ | √ | √ | √ | √ |
| 元数据加密 | × | × | √ | √ | √ |
| **DEP** |   |   |   |   |   |
| NX 支持 | √ | √ | √ | √ | √ |
| 永久DEP | × | × | √ | √ | √ |
| 默认OptOut | × | √ | × | √ | × |
| **ASLR** |   |   |   |   |   |
| PEB, TEB | √ | √ | √ | √ | √ |
| 堆 | × | × | √ | √ | √ |
| 栈 | × | × | √ | √ | √ |
| 映像 | × | × | √ | √ | √ |
| **SEHOP** |   |   |   |   |   |
 SEH链验证 | × | × | √ | √ | √ |
