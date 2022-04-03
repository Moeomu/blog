---
title: 漏洞学习笔记-006-堆的入门
description: Windows的堆，从来都是一个乱糟糟的地方，管理方法也非常奇怪
date: 2020-10-20 23:10:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

声明：实验环境为 Windows 2000

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-006-堆的入门/)

## 堆的介绍

### 与栈的区别

- 堆是由程序员使用malloc等函数向操作系统申请的一块内存空间，它能否成功与操作系统的状态有极大的关系，与管理整齐的栈不同，它的管理以及分配算法都是非常奇特的
- 堆在释放时由程序员使用free或delete释放，而栈是系统自动释放的
- 堆的地址范围变化很大，而栈的内存地址总是`0x0012XXXX`
- 堆的地址由低向高移动，栈的地址由高向低移动

### 堆的安全

- 堆是杂乱无章的，所以它的利用相对于栈会困难很多，而堆的管理微软从未公开，研究有一定困难
- 在Windows2000 - Windows XP SP1，堆管理未考虑安全因素，容易利用
- 在Windows XP SP2 - Windows 2003，加入了块首的cookie，指针验证等
- Windows Vista - Windows 7，堆管理的安全，稳定和效率都改变巨大

### 堆的数据结构和管理策略

#### 两种堆结构

- 堆块：堆区的内存按不同大小组织成块，以堆块为单位进行标识。
  - 块首：本块的大小，是否占用
  - 块身：数据区
- 堆表：位于堆区的起始位置，可索引堆区的所有重要信息，包括堆块的大小，位置，是否占用。堆表往往用不止一种数据结构来表示。
- Windows中，占用态的堆块只有占用它的程序索引，堆表只索引空闲态的堆块。
- Windows中重要的堆表：
  - 空闲双向链表：(Freelist)(空表)
    - 空表包含128个数组，第二个数组`freelist[1]`标识8字节的空堆空间，之后每项逐个递增8字节
    - `空闲堆块大小(包含堆首) = 索引项 * 8(字节)`
    - `freelist[0]`标识了所有大小大于1024字节的堆块(小于等于512KB)，它们按从小到大的顺序依次升序排列
  - 快速单向链表(Lookaside)(快表)
    - 快表包含128条数据，组织结构与空表类似，但是单链表
    - 总被初始化为空，每条快表最多4个节点
    - 每个节点都被初始化为已占用，所以不会发生堆块合并现象

#### 管理策略

- 堆块分配
  - 零号空表分配：按照大小升序链着大小不同的空闲块，从`free[0]`反向查找最后一个块，再正向搜索最小能够满足要求的空闲堆块进行分配
  - 普通空表分配：寻找最优空闲空间分配，其次找次优
  - 快表分配：寻找大小匹配的表，从堆表卸下，返回一个指向堆块的指针给程序
  - 当空表无法找到最优堆块时，一个稍大些的块会被用于分配，此为次优分配，会先从大块中按照请求的大小精确地割出一块进行分配，然后给剩下的部分重新标注块首，连入空表。
  - 快表只有精确匹配时才会分配，所以不存在以上现象
- 堆块释放
  - 将堆块状态改为空闲，链入相应的堆表。所有释放的块将链入堆表的末尾，分配的时候也先从堆表末尾拿。
- 堆块合并
  - 反复申请和释放堆区将产生很多内存碎片，为了合理有效地利用内存将合并一些堆块
  - 这个操作包含两个块从空闲链表中卸下，合并堆块，调整合并后的大块的块首信息，将新块重新链入空闲链表
  - 堆区还将进行内存紧缩(shrink the compact)由`RtlCompactHeap`执行，将对整个堆进行调整，尽量合并可用的碎片
- 堆块分配和释放的策略
  - 小块(SIZE<1KB)
    - 分配
      - 首先进行快表分配，机械能普通空表分配
      - 若失败，使用堆缓存(heap cache)分配
      - 若堆缓存分配失败，内存紧缩后尝试分配
      - 若无法分配，返回NULL
    - 释放
      - 优先链入快表(只能链入4个空闲块)
      - 若快表满，链入相应空表
  - 大块(1KB<=SIZE<512KB)
    - 分配
      - 使用堆缓存分配
      - 若堆缓存分配失败，使用free[0]中的大块进行分配
    - 释放
      - 优先将它放入堆缓存
      - 若堆缓存满，将链入freelists[0]
  - 巨块(SIZE>=512KB)
    - 分配：虚分配(并非从堆区分配)
    - 释放：直接释放，无堆表操作

---

## 实践

> 血的教训：无论是空表还是快表，它`Blink/Flink`指针指向的**永远**是下/上一个节点的`Blink/Flink`

### 测试空表

#### 代码

```CPP
#include <windows.h>

void main()
{
  HLOCAL h1, h2, h3, h4, h5, h6;
  HANDLE hp;
  hp = HeapCreate(0, 0x1000, 0x10000);
  __asm int 3;

  h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 3);
  h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 5);
  h3 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 6);
  h4 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
  h5 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 19);
  h6 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 24);

  //free block and prevent coaleses
  HeapFree(hp, 0, h1); // free to freelist[2]
  HeapFree(hp, 0, h3); // free to freelist[2]
  HeapFree(hp, 0, h5); // free to freelist[4]

  HeapFree(hp, 0, h4); // coalese h3 h4 h5 link the large block to freelist[8]
}
```

#### 观察

##### INT异常调起调试器，未运行

- `HeadCreate()`创建堆区后，将堆区指针交给EAX，观察到此时地址为`0x360000`
- 查看内存区域`0x360000`，依次向后的信息是(抄的，反正我也不知道这几个结构具体多大)段表索引(SegmentList)，虚表索引(VirtualAllocationList)，空表使用标识(freelist usage bitmap)和空表索引区
- 在偏移`0x178`处找到了空表索引，其内容是`0x00360688`，说明`freelist[0]`指向了偏移为`0x688`的地方，我们来康康这个地方存了什么
- 这个地方存了`0x00360178`，妙啊，指向了`freelist[0]`，绕了一圈指向了自己，而这个`freelist[0]`看来指向的就是唯一一个空闲的堆区，一般称为“尾块”
- 根据堆块的结构(在下面嘞)易得，实际这个堆块开始于`0x00360680`，看起来堆块指针越过了堆块块首，直接指向了数据区
- `0x1-0x2`字节是自身大小，此时这个值是`0x0130`，说明这个堆的大小是`0x130`个字节
- `0x3-0x4`字节是前一个堆块大小，这个值是`0x08`(???不是说好了唯一???)
- `0x5`字节是索引，此时为0
- `0x6`字节是Flag，此时这是1
- `0x7`是保留字节，是0
- `0x8`字节是标签索引(调试态)，不知道干啥的，是0
- `0x9-0xC`(空堆块专属)字节是前一个空堆块的地址，是`0x00360178`
- `0xD-0x10`(空堆块专属)字节是后一个空堆块的地址，同样是`0x00360178`

##### 运行六次分配

- `0x00360680-0x00360688`为h1块首，`0x00360689-0x0036068F`是8个字节的块身，内容是`00 00 00 00 78 01 36 00`
- `0x00360690-0x00360698`为h2块首，`0x00360699-0x0036069F`是8个字节的块身，内容是`00 00 00 00 00 01 36 00`
- `0x003606A0-0x003606A8`为h3块首，`0x003606A9-0x003606AF`是8个字节的块身，内容是`00 00 00 00 00 00 36 00`
- `0x003606B0-0x003606B8`为h4块首，`0x003606B9-0x003606BF`是8个字节的块身，内容是`00 00 00 00 00 00 00 00`
- `0x003606C0-0x003606C8`为h5块首，`0x003606C9-0x003606DF`是24个字节的块身，内容是`00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00`
- `0x003606E0-0x003606E8`为h5块首，`0x003606E9-0x003606FF`是24个字节的块身，内容是`00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00`

##### 释放堆

- 第一次释放的堆块大小为16字节的堆块，所以连接到了`freeList[2]`，也就是`0x188`的位置，此时内容为`0x00360688`
- 第二次释放的堆同样连接到了`freeList[2]`，细节不再描述
- 第三次释放的堆同样连接到了`freeList[2]`，细节不再描述
- 第四次释放时，h3，h4，h5相邻，所以它们合并了，其中h3h4的大小各是2个堆单位，h5则是4个，那么它们合并后共计8个堆单位，除去要存放一个堆首，它们还剩7个堆单位，所以放入`freeList[8]`

#### 结论

- 堆表中包含的信息依次是段表索引(SegmentList)，虚表索引(VirtualAllocationList)，空表使用标识(freelist usage bitmap)和空表索引区
- 当一个堆刚刚被初始化时，它的堆块状况
  - 只有一个空闲态的大块，这个块被称为“尾块”
  - 之后是快表
  - Freelist[0]指向“尾块”
  - 除了零号空表索引外，区域各项索引都指向自己，这意味着其余所有的空闲链表中都没有空闲块
- 占用态块首

```CPP
struct Flag
{
  BIT Busy;
  BIT ExtraPresent;
  BIT FillPattern;
  BIT VirtualAlloc;
  BIT LastEntry;
  BIT FFU1;
  BIT FFU2;
  BIT NoCoalesce;
}
struct BusyHeapHeadBlock
// 8 Byte Head
{
  USHORT SelfSize;
  USHORT PreviousChunkSize;
  UCHAR SegmentIndex;
  struct Flag Flags;
  UCAHR UnusedBytes;
  UCAHR TagIndex_Debug;
}

// Data After...
```

- 空闲态块首

```CPP
struct Flag
{
  BIT Busy;
  BIT ExtraPresent;
  BIT FillPattern;
  BIT VirtualAlloc;
  BIT LastEntry;
  BIT FFU1;
  BIT FFU2;
  BIT NoCoalesce;
}
struct BusyHeapHeadBlock
// 16 Byte Head
{
  USHORT SelfSize;
  USHORT PreviousChunkSize;
  UCHAR SegmentIndex;
  struct Flag Flags;
  UCAHR UnusedBytes;
  UCAHR TagIndex_Debug;
  PVOID FlinkInFreelist; // 下一个
  PVOID BlinkInFreelist; // 上一个
}

// Empty Data After...
```

- 堆块的分配
  - 堆块的大小包括了块首，所以申请32字节将会分配40字节
  - 堆块的单位是8字节，不足8字节的按照8字节分配，所以最少实际分配为16字节
  - 初始状态下，快表和空表都为空，不存在精确分配，请求将使用次优块进行分配
  - 由于次优分配的发生，分配函数将从尾块中切走一些小块，修改尾块块首的size，最后将`freelist[0]`指向新的尾块

### 测试快表

#### 代码

```CPP
#include <stdio.h>
#include <windows.h>

void main()
{
  HLOCAL h1, h2, h3, h4;
  HANDLE hp;
  hp = HeapCreate(0, 0, 0);
  __asm int 3
  h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
  h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
  h3 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 16);
  h4 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 24);
  HeapFree(hp, 0, h1);
  HeapFree(hp, 0, h2);
  HeapFree(hp, 0, h3);
  HeapFree(hp, 0, h4);
  h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 16);
  HeapFree(hp, 0, h2);
}
```

#### 结论

- 块首标识位为0x01
- 只存指向下一堆块的指针，不存在指向前一堆块的指针
- 偏移`0x178`处的`freeList[0]`的地址变为了`0x00361E90`，原本的`0x00360688`被快表霸占了
- 快表从`0x688`开始，每个结构共`0x30`个字节，前四个字节的内容是快表单链表
- 虽然0Day安全书中写道，8字节的堆区被插入`lookaside[1]`，但是我似乎觉得，`0x688`处的才是`lookaside[0]`，`0x6B8`处的才是`lookaside[1]`，而`0x0E8`处的可以被称为`lookaside[2]`，它存放的堆块带上块首的大小一共16个字节
