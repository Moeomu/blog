---
title: 漏洞利用学习笔记-021-保护下的堆
description: 堆的保护和攻击
date: 2020-11-28 16:15:00+0800
categories:
    - 漏洞利用
tags:
    - Windows
    - 堆保护
---

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞利用学习笔记-021-保护下的堆/)

## 简介

- PEB Random：微软在`Windows XP SP2`之后不再使用固定的PEB基址`0x7ffdf000`，而是使用具有一定随机性的PEB基址。PEB随机化之后主要影响了对PEB中函数的攻击，在`DWORD SHOOT`的时候，PEB中的函数指针是绝佳的目标，移动PEB基址将在一定程度上给这类攻击增加难度。覆盖PEB中函数指针的利用方式请参见[堆溢出利用](https://www.moeomu.com/posts/%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-007-%E5%A0%86%E6%BA%A2%E5%87%BA%E7%9A%84%E5%88%A9%E7%94%A8/)中的实验和[攻击PEB中的函数指针](https://www.moeomu.com/posts/%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-007-%E5%A0%86%E6%BA%A2%E5%87%BA%E7%9A%84%E5%88%A9%E7%94%A8/)的相关介绍
- `SafeUnlink`：微软改写了操作双向链表的代码，在卸载`free list`中的堆块时更加小心。对照[堆溢出利用-DWORD SHOOT](https://www.moeomu.com/posts/%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-007-%E5%A0%86%E6%BA%A2%E5%87%BA%E7%9A%84%E5%88%A9%E7%94%A8/)中关于双向链表拆卸问题的描述，在SP2之前的链表拆卸操作类似于如下代码：

```cpp
int remove(ListNode * node)
{
    node -> blink -> flink = node -> flink;
    node -> flink -> blink = node -> blink;
    return 0;
}
```

- SP2 在进行删除操作时，将提前验证堆块前向指针和后向指针的完整性，以防止发生`DWORD SHOOT`：

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

- heap cookie：栈中的`security cookie`类似，微软在堆中也引入了cookie，用于检测堆溢出的发生。cookie被布置在堆首部分原堆块的`segment table`的位置，占1个字节大小

![heap struct](./heap%20struct.png)

- 元数据加密：微软在`Windows Vista`及后续版本的操作系统中开始使用该安全措施。块首中的一些重要数据在保存时会与一个4字节的随机数进行异或运算，在使用这些数据时候需要再进行一次异或运行来还原，这样我们就不能直接破坏这些数据了，以达到保护堆的目的。

## 攻击思路

### 攻击堆内存储的变量

> 这样的方法是攻击堆内存储的函数指针之类的方法实现溢出，但是和堆本身并没有什么关系

### 利用chunk重设大小攻击堆

#### 原理

> `SafeUnlink`在堆从freelist中卸下堆块的时候进行双链表有效性校验，但是将堆块插入freelist中的操作却没有校验

#### 时机

- 内存中释放堆块后，将会被插入空表
- 堆块的空间大于申请的空间，剩余的空间将被插入空表

### 新chunk的插入过程

> Flink：下一个节点；Blink：上一个节点；参见[MSDN-NTDEF-LIST](https://docs.microsoft.com/en-us/windows/win32/api/ntdef/ns-ntdef-list_entry)

- 新chunk->Blink=旧chunk->Flink->Blink
- 旧chunk->Flink->Blink->Flink=新chunk
- 旧chunk->Flink->Blink=新chunk

> 只要将旧chunk的Flink指针覆盖为地址，将Blink覆盖为值，就又可以进行DWORDSHOOT了

### 代码

```cpp

```

### 未完待续
