---
title: Windows内核学习笔记-002
description: 这一节依旧是基础内容的复习
date: 2020-12-18 20:52:00+0800
categories:
    - Kernel
tags:
    - Windows
    - Kernel
---

本文来源：[Moeomu的博客](/zh-cn/posts/Windows内核学习笔记-002/)

## 字符串操作

> 内核中使用UNICODE_STRING结构来作为基本的字符串结构，应该注意的是使用此结构中的lenth成员确定字符串长度，而不是`'\0'`。

### 字符串初始化

- 函数：`RtlInitUnicodeString`
- 参数：
  - `PUNICODE_STRING`: `DestinationString`
  - `PCWSTR`: `SourceString`
- 返回值：无
- IRQL：`<=DISPATCH_LEVEL`
- 解释：初始化一个以0结尾的WCHAR字符串，第一个参数是输入参数也是输出参数

```C
UNICODE_STRING uFirstString = {0};
RtlInitUnicodeString(&uFirstString, L"HelloWorld\n");
DbgPrint("String:%wZ", &uFirstString);
```

> 坑：它并没有为buffer分配空间，而是直接指向Source首地址，因此要保证Source始终有效，否则就是无效访问

### 字符串拷贝

- 函数：`RtlUnicodeStringCopyString`
- 参数：
  - `PUNICODE_STRING`: `DestinationString`
  - `NTSTRSAFE_PCWSTR`: `pszSrc`
- 返回值：`NTSTAUTS`
  - 成功执行返回`STATUS_SUCCESS`
- IRQL：`=PASSIVE_LEVEL`
- 解释：将src复制一份到dest里面

```C
WCHAR strBuf[128] = {0};
UNICODE_STRING uFirstString = {0};
RtlInitEmptyUnicodeString(&uFirstString, strBuf, sizeof(strBuf));
RtlUnicodeStringCopyString(&uFirstString, L"Hello Kernel\n");
DbgPrint("String: %wZ", &uFirstString);
```

> 坑：为了使用RtlUnicodeStringCopyString函数，应该添加头文件`Ntstrsafe.h`；不能copy到定长buf的String里，否则会蓝屏报内存读写错误

## 链表

### 链表的定义

> 以下是wdk中链表的定义

```C
typedef struct _LIST_ENTRY
{
    struct _LIST_ENTRY *Flink; // 后节点
    struct _LIST_ENTRY *Blink; // 前节点
} LIST_ENTRY, *PLIST_ENTRY;
```

### 使用链表

```C
typedef struct _TestListEntry
{
    ULONG m_ulData1;
    ULONG m_ulData2;
    LIST_ENTRY m_ListEntry;
    ULONG m_ulData3;
    ULONG m_ulData4;
}
```

- 一般而言，为了方便操作，会定义一个链表的头结点，不包含任何内容，只是一个LIST_ENTRY结构。

### 头节点初始化

```C
LIST_ENTRY ListHeader = {0};
InitializeListHead(&ListHeader);
```

### 节点插入

```C
LIST_ENTRY ListHeader = {0};
TestListEntry Entry1 = {0};
TestListEntry Entry2 = {0};
TestListEntry Entry3 = {0};

Entry1.m_ulData1 = 'A';
Entry2.m_ulData1 = 'B';
Entry3.m_ulData1 = 'C';

InitializeListHead(&ListHeader);
InsertHeadList(&ListHeader, &Entry2.m_ListEntry);
InsertHeadList(&ListHeader, &Entry1.m_ListEntry);
InsertTailList(&ListHeader, &Entry3.m_ListEntry);
```

### 链表遍历

```C
PLIST_ENTRY pListEntry = NULL;
pListEntry = ListHeader.Flink;
while(pListEntry != &ListHeader)
{
    PTestListEntry pTestEntry = CONTAINING_RECORD(pListEntry, TestListEntry, m_ListEntry);
    DbgPrint("ListPtr=%p, Entry=%p, Tag=%c\n", pListEntry, pTestEntry, (CHAR)pTestEntry->m_ulData1);
    pListEntry = pListEntry->Flink;
}
```

- `CONTAINING_RECORD`的作用是将`m_ListEntry`地址转为结构体`TestListEntry`首地址
- `CONTAINING_RECORD`用法：`CONTAINING_RECORD(PCHAR Address, TYPE Type, PCHAR Field)`

### 节点移除

- 移除首节点：`PLIST_ENTRY RemoveHeadList(PLIST_ENTRY ListHead)`
- 移除尾节点：`PLIST_ENTRY RemoveTailList(PLIST_ENTRY ListHead)`
- 如果成功，上面两个函数都将返回链首地址，如果无法移除，则返回NULL
- 移除特定节点：
  - `BOOLEAN RemoveEntryList(PLIST_ENTRY Entry)`
  - 若移除后变成空链表，那么将返回TRUE，如果非空，则返回FALSE

### 判断链表状态

- `BOOLEAN IsListEmpty(const LIST_ENTRY *ListHead)`
- 它返回TRUE时表示为空链表，否则表示链表非空

## 自旋锁

### 使用自旋锁

> 自旋锁是内核提供的一种高IRQL锁，用同步以及独占的方式访问某个资源
>
> **注意事项**：  
>
> - 自旋锁变量不能存放在当前函数栈中，否则每次进入初始化一遍跟不初始化一样

### 初始化/使用自旋锁

```C
// Initialize Spin Lock WARN: not local var
KSPIN_LOCK my_spin_lock;
void initLock()
{
  KeInitializeSpinLock(&my_spin_lock);
}
void TestFuncLock()
{
  // it's a safe function

  // Acquire Lock
  KIRQL irql; // save old irql

  // Normal Spin Lock
  KeAcquireSpinLock(&my_spin_lock, &irql);
  // TO DO
  KeReleaseSpinLock(&my_spin_lock, irql);
}
```

### 自旋锁在双向链表中使用

```C
void TestFuncLock()
{
    // it's a safe function
    DbgPrint("[%ws] Enter...\n", __FUNCTIONW__);
    // Acquire Lock
    KIRQL irql; // save old irql

    // Normal Spin Lock
    KeAcquireSpinLock(&my_spin_lock, &irql);
    // Test List
    typedef struct _FILE_INFO
    {
        LIST_ENTRY m_ListEntry;
        UNICODE_STRING m_strFileName;
    }FILE_INFO, *PFILE_INFO;

    LIST_ENTRY listHead;
    FILE_INFO my_file_info;
    RtlInitUnicodeString(&my_file_info.m_strFileName, L"TestName");

    InitializeListHead(&listHead);
    ExInterlockedInsertHeadList(&listHead, &my_file_info.m_ListEntry, &my_spin_lock);
    KeReleaseSpinLock(&my_spin_lock, irql);
}
```

### 队列自旋锁

> 队列自旋锁可以在多CPU平台上有更好的性能，也遵循先等待先获取自旋锁的原则。

- 它和普通自旋锁的初始化是一样的，但是初始化后的自旋锁绝不能混用

```C
KSPIN_LOCK my_spin_lock;
void initLock()
{
  KeInitializeSpinLock(&my_spin_lock);
}

void TestFuncLock()
{
    // it's a safe function

    // Acquire Lock
    KIRQL irql; // save old irql

    // Queue Spin Lock
    KLOCK_QUEUE_HANDLE my_lock_queue_handle;
    KeAcquireInStackQueuedSpinLock(&my_spin_lock, &my_lock_queue_handle);
    KeReleaseInStackQueuedSpinLock(&my_lock_queue_handle);
}
```

## 内存分配

### 常规内存分配

```C
void TestFuncMem()
{
    PVOID buffer = ExAllocatePoolWithTag(NonPagedPoolNx, 512, 'tag1');
    if (buffer)
    {
        ExFreePoolWithTag(buffer, 'tag1');
        DbgPrint("[%ws] Pool Operate Success!\n", __FUNCTIONW__);
    }
    else
    {
        DbgPrint("[%ws] Allocate Pool Failed!\n", __FUNCTIONW__);
    }
}
```

### 快表内存分配

> 优点：高频率从系统中申请和释放内存，使用快表分配将大大提高性能

- 注意：有些地方称为“旁视列表”(LookAside)

```C
void TestFuncMemLookaside()
{
    PNPAGED_LOOKASIDE_LIST pLookAsideList = NULL;
    BOOLEAN bSucc = FALSE;
    BOOLEAN bInit = FALSE;
    PVOID pFirstMemory = NULL;
    PVOID pSeocdeMemory = NULL;

    do 
    {
        pLookAsideList = (PNPAGED_LOOKASIDE_LIST)ExAllocatePoolWithTag(NonPagedPoolNx, sizeof(NPAGED_LOOKASIDE_LIST), 'test');
        if (pLookAsideList == NULL)
        {
            break;
        }
        memset(pLookAsideList, 0, sizeof(NPAGED_LOOKASIDE_LIST));

        // init
        ExInitializeNPagedLookasideList(pLookAsideList, NULL, NULL, 0, 128, 'test', 0);
        bInit = TRUE;

        // start allocate
        pFirstMemory = ExAllocateFromNPagedLookasideList(pLookAsideList);
        if (pFirstMemory == NULL)
        {
            break;
        }
        pSeocdeMemory = ExAllocateFromNPagedLookasideList(pLookAsideList);
        if (pSeocdeMemory == NULL)
        {
            break;
        }
        DbgPrint("[%ws] First Address:%p, Second Address:%p\n", __FUNCTIONW__, pFirstMemory, pSeocdeMemory);
        
        // free first
        ExFreeToNPagedLookasideList(pLookAsideList, pFirstMemory);
        pFirstMemory = NULL;

        // reallocate
        pFirstMemory = ExAllocateFromNPagedLookasideList(pLookAsideList);
        if (pFirstMemory == NULL)
        {
            break;
        }
        DbgPrint("[%ws] Re-Allocate First Address:%p\n", __FUNCTIONW__, pFirstMemory);
        bSucc = TRUE;
    } while (FALSE);

    if (pFirstMemory != NULL)
    {
        ExFreeToNPagedLookasideList(pLookAsideList, pFirstMemory);
        pFirstMemory = NULL;
    }
    if (pSeocdeMemory != NULL)
    {
        ExFreeToNPagedLookasideList(pLookAsideList, pSeocdeMemory);
        pSeocdeMemory = NULL;
    }
    if (bInit == TRUE)
    {
        ExDeleteNPagedLookasideList(pLookAsideList);
        bInit = FALSE;
    }
    if (pLookAsideList != NULL)
    {
        ExFreePoolWithTag(pLookAsideList, 'test');
        pLookAsideList = NULL;
    }
}
```

## 对象与句柄

> 在内核中创建，在内核中销毁，由内核管理与维护的对象称为内核对象

```C
void TestFuncObject()
{
    BOOLEAN bSucc = FALSE;
    HANDLE hCreateEvent = NULL;
    PVOID pCreateEventObject = NULL;
    HANDLE hOpenEvent = NULL;
    PVOID pOpenEventObject = NULL;

    do 
    {
        OBJECT_ATTRIBUTES ObjAttr = { 0 };
        UNICODE_STRING uNameString = { 0 };
        RtlInitUnicodeString(&uNameString, L"\\BaseNamedObjects\\TestEvent");
        InitializeObjectAttributes(&ObjAttr, &uNameString, OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE, NULL, NULL);
        ZwCreateEvent(&hCreateEvent, EVENT_ALL_ACCESS, &ObjAttr, SynchronizationEvent, FALSE);
        if (hCreateEvent == NULL)
        {
            break;
        }
        // get point
        ObReferenceObjectByHandle(hCreateEvent, EVENT_ALL_ACCESS, *ExEventObjectType, KernelMode, &pCreateEventObject, NULL);
        if (pCreateEventObject == NULL)
        {
            break;
        }
        // open obj with attribute:name
        ZwOpenEvent(&hOpenEvent, EVENT_ALL_ACCESS, &ObjAttr);
        if (hOpenEvent == NULL)
        {
            break;
        }
        ObReferenceObjectByHandle(hOpenEvent, EVENT_ALL_ACCESS, *ExEventObjectType, KernelMode, &pOpenEventObject, NULL);
        if (pOpenEventObject == NULL)
        {
            break;
        }
        DbgPrint("[%ws] Create Handle:%p, Create Object Address:%p\n", __FUNCTIONW__, hCreateEvent, pCreateEventObject);
        DbgPrint("[%ws] Open Handle:%p, Open Object Address:%p\n", __FUNCTIONW__, hOpenEvent, pOpenEventObject);
        bSucc = TRUE;
    } while (FALSE);

    if (pCreateEventObject == NULL)
    {
        ObDereferenceObject(pCreateEventObject);
        pCreateEventObject = NULL;
    }
    if (hCreateEvent == NULL)
    {
        ZwClose(hCreateEvent);
        hCreateEvent = NULL;
    }
    if (pOpenEventObject == NULL)
    {
        ObDereferenceObject(pOpenEventObject);
        pOpenEventObject = NULL;
    }
    if (hOpenEvent == NULL)
    {
        ZwClose(hOpenEvent);
        hOpenEvent = NULL;
    }
}
```

> 一个小小的坑：在导入头文件的时候发生冲突：`ntddk.h`和`ntifs.h`，解决办法是将`ntifs.h`放到`ntddk.h`的前面导入，这样就没有冲突了

## 注册表

> 注册表其实是Windows的配置存储结构，存储着系统绝大部分的配置信息，其中大多数文件存储在系统盘下SYSTEM32\CONFIG目录下，这些文件以内存映射方式存储在内核空间中，然后以“HIVE”的方式组织起来，注册表API实际上操作的是HIVE内存数据，最终会写回到config目录下对应的文件中去

### 打开与关闭

- 未完待续

```C

```
