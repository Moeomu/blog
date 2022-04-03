---
title: Windows内核学习笔记-001
description: 这波啊，这波是又从头学起了，之前学的内核有些不系统，这次一定系统地整理一遍Windows内核编程的内容
date: 2020-12-18 18:52:00+0800
categories:
    - Kernel
tags:
    - Windows
    - Kernel
---

本文来源：[Moeomu的博客](/zh-cn/posts/windows内核学习笔记-001/)

## Windows内核开发环境配置

### 下载

> 开发机

- Windows 10 20H2 x64
- Visual Studio 2019
- Windows Driver Kit - Windows 10.0.19041.685(Windows 10 2004)
- WinDbg Preview

> 测试机

- Windows 10 2004 x64
- DbgView

### 测试驱动程序

> 还是使用最经典的HelloWorld来作为开始吧

```C
#include <ntddk.h>

VOID DriverUnload(PDRIVER_OBJECT DriverObject)
{
    DbgPrint("[%ws] Driver Unload, Driver Object Address: %p\n", __FUNCTIONW__, DriverObject);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    DbgPrint("[%ws] Hello Kernel World!\n", __FUNCTIONW__);
    if (DriverObject != NULL)
    {
        DbgPrint("[%ws] Driver Object Address: %p\n", __FUNCTIONW__, DriverObject);
        DriverObject->DriverUnload = DriverUnload;
    }
    if (RegistryPath != NULL)
    {
        DbgPrint("[%ws] Driver Registry Path: %wZ\n", __FUNCTIONW__, RegistryPath);
    }

    return STATUS_SUCCESS;
}
```

### 测试调试

> 参考[Windows内核调试](/posts/Windows%E5%86%85%E6%A0%B8%E8%B0%83%E8%AF%95%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-001-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)一文来布置调试环境，只不过将Windows7的地方改成Windows10即可

## 上下文环境解析

> 上下文指的是CPU在执行代码时，改代码所处的环境与状态。

### 实验：PsGetCurrentProcessId

> 实验目的是为了找出所写出的驱动模块究竟在哪个“进程”中执行

```C
#include <ntddk.h>

VOID DriverUnload(PDRIVER_OBJECT DriverObject)
{
    DbgPrint("[%ws] Driver Unload, Driver Object Address: %p, Current Process Id=0x%p\n", __FUNCTIONW__, DriverObject, PsGetCurrentProcessId());
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    DbgPrint("[%ws] Hello Kernel World, Current Process Id=0x%p\n", __FUNCTIONW__, PsGetCurrentProcessId());
    if (DriverObject != NULL)
    {
        DbgPrint("[%ws] Driver Object Address: %p\n", __FUNCTIONW__, DriverObject);
        DriverObject->DriverUnload = DriverUnload;
    }
    if (RegistryPath != NULL)
    {
        DbgPrint("[%ws] Driver Registry Path: %wZ\n", __FUNCTIONW__, RegistryPath);
    }

    return STATUS_SUCCESS;
}
```

- 如图

![rYh5BF.png](https://s3.ax1x.com/2020/12/18/rYh5BF.png)
![rYhq91.png](https://s3.ax1x.com/2020/12/18/rYhq91.png)

### 结论

> 无论是驱动入口函数还是驱动卸载回调函数都隶属于ID为4的进程，此进程为System进程

- System进程是操作系统虚拟出来的一个进程，代表系统内核
- 驱动和应用层上下文应当严格区分，若进程A运行在P1虚拟空间内，而驱动当前的CPU上下文是P2的虚拟空间，那么访问到的内容应当是不可预料的

## 中断请求级别

> 与线程的优先级的概念类似，系统调度器以时间片为粒度调度，根据线程的优先级来调度线程，线程优先级越高，获得调度的机会越大。而在驱动层，CPU提供了IRQL的概念，规定高IRQL级别的代码可以中断、抢占低IRQL的代码的执行过程，从而执行。

### 常见的IRQL中断请求级别表

| IRQL | 数值(x86, amd64, IA64) | 描述 |
|-|-|-|
| PASSIVE_LEVEL | 0, 0, 0 | 应用层线程以及大部分内核函数处于该IRQL，可与无限制使用所有内核API，可以访问分页以及非分页内存 |
| APC_LEVEL | 1, 1, 1 | 异步方法调用(APC)，或者页错误时处于该IRQL，可以使用大部分内核API，可以访问分页以及非分页内存 |
| DISPATCH_LEVEL | 2, 2, 2 | 延迟方法调用(DPC)时处于该IRQL，可以使用特定的内核API，只能访问非分页内存 |

### 判断当前IRQL

- 在驱动程序入口点DriverEntry，IRQL为PASSIVE_LEVEL，这是系统保证的
- 通过调用KeGetCurrentIrql函数来获取当前的IRQL
- 如图所示，IRQL都是0，对照上表，级别是PASSIVE_LEVEL

![rYHLIU.png](https://s3.ax1x.com/2020/12/18/rYHLIU.png)]

### 结论

- 在调用某个函数之前首先看清楚函数说明文档，仔细观察安全调用函数的IRQL级别是什么，以此来实现安全编程

## 驱动异常

> 开发驱动时，若驱动程序代码编写不合规引发系统崩溃的情况，表现为蓝屏(BSOD)。

### 常见原因

- 高IRQL死锁
- 内存访问违规
- 函数堆栈不平衡

### 主动引发蓝屏

- 可以使用KeBugCheckEx函数主动引发蓝屏

### 结论

- 在代码发生不可预知错误时，主动引发蓝屏可以减少进一步扩大错误
