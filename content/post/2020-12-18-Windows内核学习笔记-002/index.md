---
title: Windows Kernel Learning Notes 002
description: This section is still a review of the basics
date: 2020-12-18 20:52:00+0800
categories:
    - Kernel
tags:
    - Windows
    - Kernel
---

Source: [Moeomu's blog](/posts/windows-kernel-learning-notes-002/)

## String manipulation

> The UNICODE_STRING structure is used in the kernel as the basic string structure. It should be noted that the lenth member of this structure is used to determine the string length, not `'\0'`.

### String initialization

- Function: `RtlInitUnicodeString`
- Parameters.
  - `PUNICODE_STRING`: `DestinationString`
  - `PCWSTR`: `SourceString`
- Return value: None
- IRQL: `<=DISPATCH_LEVEL`
- Explanation: Initialize a WCHAR string ending with 0, the first parameter is the input parameter and also the output parameter

```C
UNICODE_STRING uFirstString = {0};
RtlInitUnicodeString(&uFirstString, L"HelloWorld\n");
DbgPrint("String:%wZ", &uFirstString);
```

> ps: it does not allocate space for buffer, but points directly to Source first address, so make sure Source is always valid, otherwise it is invalid access

### String Copy

- Function: `RtlUnicodeStringCopyString`
- Parameters
  - `PUNICODE_STRING`: `DestinationString`
  - `NTSTRSAFE_PCWSTR`: `pszSrc`
- Return value: `NTSTAUTS`
  - Successful execution returns `STATUS_SUCCESS`
- IRQL: `=PASSIVE_LEVEL`
- Explanation: Copy a copy of src to dest

```C
WCHAR strBuf[128] = {0};
UNICODE_STRING uFirstString = {0};
RtlInitEmptyUnicodeString(&uFirstString, strBuf, sizeof(strBuf));
RtlUnicodeStringCopyString(&uFirstString, L"Hello Kernel\n");
DbgPrint("String: %wZ", &uFirstString);
```

> PS: In order to use the RtlUnicodeStringCopyString function, you should add the header file `Ntstrsafe.h`; you can't copy to the String with fixed length buf, otherwise you will blue screen report memory read/write error

## Chain table

### Definition of a linked table

> The following is the definition of a linked table in wdk

```C
typedef struct _LIST_ENTRY
{
    struct _LIST_ENTRY *Flink; // 后节点
    struct _LIST_ENTRY *Blink; // 前节点
} LIST_ENTRY, *PLIST_ENTRY;
```

### Using linked tables

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

- Generally, for ease of operation, a header node of a chain table is defined, containing nothing but a LIST_ENTRY structure.

### Header node initialization

```C
LIST_ENTRY ListHeader = {0};
InitializeListHead(&ListHeader);
```

### Node insertion

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

### Link table traversal

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

- The role of `CONTAINING_RECORD` is to convert the address of `m_ListEntry` to the first address of the structure `TestListEntry`.
- `CONTAINING_RECORD` usage: `CONTAINING_RECORD(PCHAR Address, TYPE Type, PCHAR Field)`

### Node Removal

- Remove the first node: `PLIST_ENTRY RemoveHeadList(PLIST_ENTRY ListHead)`
- Remove the tail node: `PLIST_ENTRY RemoveTailList(PLIST_ENTRY ListHead)`
- If successful, both of the above functions will return the address of the head of the chain, or NULL if they cannot be removed
- To remove a specific node.
  - `BOOLEAN RemoveEntryList(PLIST_ENTRY Entry)`
  - If the chain becomes empty after removal, then TRUE will be returned, if it is not empty, then FALSE will be returned

### Determine the state of the linked list

- `BOOLEAN IsListEmpty(const LIST_ENTRY *ListHead)`
- It returns TRUE to indicate an empty linked table, otherwise it means the chain is non-empty

## Spin locks

### Using spin locks

> A spinlock is a high IRQL lock provided by the kernel to access a resource in a synchronous and exclusive manner
>
> **Caution**.  
>
> - The spinlock variable cannot be stored on the current function stack, otherwise it is the same as not initializing it every time you enter it

### Initializing/using spin locks

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

### Spin locks are used in bidirectional linked tables

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

### Queue spinlock

> Queue spinlock can have better performance on multi-CPU platforms, and also follows the first-wait-first-acquire spinlock principle.

- It is initialized in the same way as a normal spinlock, but the initialized spinlocks must not be mixed

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

## Memory allocation

### General memory allocation

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

### Lookaside Memory Allocation

> Benefits: High frequency of memory requests and releases from the system, using Lookaside allocation will greatly improve performance

- Note: In some places it is called "LookAside".

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

## Objects and handles

> Objects created in the kernel, destroyed in the kernel, and managed and maintained by the kernel are called kernel objects

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

> PS: there is a conflict when importing header files: `ntddk.h` and `ntifs.h`, the solution is to put `ntifs.h` in front of `ntddk.h` and import it, so there is no conflict

## Registry

> The registry is actually the configuration storage structure of Windows, storing most of the system configuration information, most of the files are stored in the SYSTEM32\CONFIG directory under the system disk, these files are stored in the kernel space in a memory-mapped way, and then organized in the way of "HIVE". The registry API actually manipulates the HIVE memory data, which is eventually written back to the corresponding file in the config directory

### Open and close

- To be continued

```C

```
