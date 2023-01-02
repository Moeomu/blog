---
title: Windows Kernel Programming Study Notes 001 Environment Building
description: This time is to learn from scratch again, before learning the kernel some unsystematic, this time must systematically organize the content of the Windows kernel programming again
date: 2020-12-18 18:52:00+0800
categories:
    - KernelProgramming
tags:
    - Windows
    - Kernel
---

Source: [Moeomu's blog](/posts/windows-kernel-programming-study-notes-001-environment-building/)

## Windows kernel development environment configuration

### Download

> Development Machine

- Windows 10 20H2 x64
- Visual Studio 2019
- Windows Driver Kit - Windows 10.0.19041.685 (Windows 10 2004)
- WinDbg Preview

> Test Machine

- Windows 10 2004 x64
- DbgView

### Test driver

> Let's start with the classic HelloWorld

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

### Test debugging

> Refer to [Windows kernel debugging](/posts/Windows%E5%86%85%E6%A0%B8%E8%B0%83%E8%AF%95%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-001-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/) article to set up the debugging environment, except that the place of Windows 7 can be changed to Windows 10

## Contextual environment analysis

> Context refers to the environment and state that the CPU is in when executing the code and changing the code.

### Experiment: PsGetCurrentProcessId

> The purpose of the experiment is to find out in which "process" the written driver module is executed

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

- As shown

![rYh5BF.png](https://s3.ax1x.com/2020/12/18/rYh5BF.png)
![rYhq91.png](https://s3.ax1x.com/2020/12/18/rYhq91.png)

### Conclusion

> Both the driver entry function and the driver uninstall callback function belong to the process with ID 4, and this process is the System process

- System process is a process virtualized by the operating system, representing the system kernel
- If process A is running in P1 virtual space and the current CPU context of the driver is P2 virtual space, then the accessed content should be unpredictable

## Interrupt request level

> Similar to the concept of priority of threads, the system scheduler schedules threads at the granularity of time slice, based on their priority, the higher the thread priority, the higher the chance of getting scheduled. And at the driver level, the CPU provides the concept of IRQL, which stipulates that code at high IRQL level can interrupt and preempt the execution process of code at low IRQL to execute.

### Table of common IRQL interrupt request levels

| IRQL | Value(x86, amd64, IA64) | Description |
|-|-|-|
| PASSIVE_LEVEL | 0, 0, 0 | Application layer threads and most kernel functions are in this IRQL, with unlimited access to all kernel APIs, paged and non-paged memory |
| APC_LEVEL | 1, 1, 1 | Asynchronous method calls (APC), or being in this IRQL on page errors, can use most of the kernel APIs and can access paged as well as non-paged memory |
| DISPATCH_LEVEL | 2, 2, 2 | Deferred method calls (DPCs) are in this IRQL, can use specific kernel APIs, and can only access non-paged memory |

### Determine the current IRQL

- At the driver entry point DriverEntry, IRQL is PASSIVE_LEVEL, which is guaranteed by the system
- Get the current IRQL by calling KeGetCurrentIrql function
- As shown in the figure, the IRQL are 0, against the above table, the level is PASSIVE_LEVEL

![rYHLIU.png](https://s3.ax1x.com/2020/12/18/rYHLIU.png)]

### Conclusion

- Before calling a certain function first read the function description document and carefully observe what the IRQL level of the safe calling function is, so as to achieve safe programming

## Driver exceptions

> When developing a driver, if the driver code is not written in compliance with the situation that triggers a system crash, manifested as a blue screen (BSOD).

### Common causes

- High IRQL deadlock
- Memory access violation
- Function stack imbalance

### Active blue screen triggered

- Blue screen can be triggered proactively using KeBugCheckEx function

### Conclusion

- Proactively raising a blue screen in case of unpredictable errors in code can reduce further expansion of the error

