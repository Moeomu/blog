---
title: Exploit learning notes 021 Protected HEAP
description: heap protection and attacks
date: 2020-11-28 16:15:00+0800
categories:
    - Exploit
tags:
    - Windows
    - ProtectedHeap
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-021-protected-heap/)

## Introduction

- PEB Random: Microsoft no longer uses a fixed PEB base address `0x7ffdf000` after `Windows XP SP2`, but a PEB base address with some randomness. the PEB randomization mainly affects attacks on functions in the PEB, and function pointers in the PEB are excellent targets when `DWORD SHOOT`. Moving the PEB base address will make such attacks more difficult to some extent. See [heap overflow exploitation](https://www.moeomu.com/posts/%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-007-%E5%A0%86%E6%BA%A2%E5%87%BA%E7%9A%84%E5%88%A9%E7%94%A8/) and [Attacking function pointers in the PEB](https://www.moeomu.com/posts/%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-007-%E5%A0%86%E6%BA%A2%E5%87%BA%E7%9A%84%E5%88%A9%E7%94%A8/) of the related introduction
- `SafeUnlink`: Microsoft has rewritten the code for manipulating bidirectional chained tables to be more careful when unloading heap blocks in `free list`. Compare to [heap overflow exploit-DWORD SHOOT](https://www.moeomu.com/posts/%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-007-%E5%A0%86%E6%BA%A2%E5%87%BA%E7%9A%84%E5%88%A9%E7%94%A8/) for the description of the bi-directional chained table disassembly problem, the chain table disassembly operation before SP2 was similar to the following code.

```cpp
int remove(ListNode * node)
{
    node -> blink -> flink = node -> flink;
    node -> flink -> blink = node -> blink;
    return 0;
}
```

- SP2 will verify the integrity of the heap block forward and backward pointers in advance when performing a delete operation to prevent `DWORD SHOOT`.

```cpp
int safe_remove(ListNode * node)
{
    if((node->blink->flink == node) && (node->flink->blink == node))
    {
        node -> blink -> flink = node -> flink;
        node -> flink -> blink = node -> blink;
        return 1;
    }
    else
    {
        // 链表指针被破坏，进入异常
        return 0;
    }
}
```

- heap cookie: similar to the `security cookie` in the stack, Microsoft has introduced a cookie in the heap to detect the occurrence of heap overflows. cookies are placed at the location of the `segment table` of the original heap block at the head of the heap and occupy a size of 1 byte

![heap struct](https://s3.ax1x.com/2020/11/28/DyrdgS.png)

- Metadata encryption: Microsoft started using this security measure in `Windows Vista` and subsequent versions of the operating system. Some important data in the head of the block will be saved with a 4-byte random number to perform an iso operation, when using these data need to perform another iso run to restore, so that we can not directly destroy these data to protect the heap.

## Attack ideas

### Attacking the variables stored inside the heap

> This is a way to achieve overflow by attacking function pointers stored in the heap or something like that, but it doesn't have anything to do with the heap itself

### Attacking the heap using chunk resizing

#### Principle

> `SafeUnlink` checks for double-linked table validity when the heap is unloaded from the freelist, but the insertion of a heap chunk into the freelist is not checked

#### Timing

- When the heap block is released from memory, it will be inserted into the empty table
- If the heap block has more space than the requested space, the remaining space will be inserted into the empty table

### The insertion process of new chunk

> Flink: next node; Blink: previous node; see [MSDN-NTDEF-LIST](https://docs.microsoft.com/en-us/windows/win32/api/ntdef/ns-ntdef-list_entry)

- New chunk->Blink = old chunk->Flink->Blink
- old chunk->Flink->Blink->Flink=new chunk
- Old chunk->Flink->Blink=New chunk

> Just overwrite the Flink pointer of the old chunk with the address and overwrite Blink with the value, and you're ready to DWORDSHOOT again

### Code

```cpp

```

### Unfinished business
