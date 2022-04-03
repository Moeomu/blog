---
title: Exploit Learning Notes 006 Heap Start
description: The Windows heap has always been a messy place and managed in a very strange way
date: 2020-10-20 23:10:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

Disclaimer: The experimental environment is Windows 2000

Source: [Moeomu's blog](/posts/exploit-learning-notes-006-heap-start/)

## Introduction to heap

### Difference with stack

- A heap is a piece of memory space requested from the operating system by the programmer using functions such as malloc. Whether it succeeds or not has a great deal to do with the state of the operating system, unlike the neatly managed stack, its management and allocation algorithm are very peculiar
- The heap is freed by the programmer using free or delete, while the stack is automatically freed by the system.
- The address range of the heap varies greatly, while the memory address of the stack is always `0x0012XXXX`.
- Heap addresses move from low to high and stack addresses move from high to low

### Heap safety

- The heap is cluttered, so its use will be much more difficult compared to the stack, and the management of the heap has never been disclosed by Microsoft, so it is difficult to study
- In Windows 2000 - Windows XP SP1, heap management does not take security into account and is easy to exploit
- In Windows XP SP2 - Windows 2003, cookies and pointer validation at the head of the block were added.
- Windows Vista - Windows 7, the security, stability and efficiency of heap management have changed dramatically

### Heap data structures and management policies

#### Two types of heap structures

- Heap blocks: The memory in the heap area is organized into blocks of different sizes, identified by heap blocks.
  - Block head: the size of this block, whether it is occupied or not
  - Block body: data area
- Heap table: Located at the beginning of the heap area, it can index all important information about the heap area, including the size, location, and occupancy or not of the heap block. The heap table is often represented by more than one data structure.
- In Windows, heap blocks in the occupied state are only indexed by the program that occupies them, and the heap table only indexes heap blocks in the idle state.
- Important heap tables in Windows.
  - Idle bidirectional linked table: (Freelist) (Empty table)
    - The empty table contains 128 arrays, the second array `freelist[1]` identifies 8 bytes of empty heap space, after which each item is incremented by 8 bytes one by one
    - `free heap block size (including heap head) = index item * 8(bytes)`
    - `freelist[0]` identifies all heap blocks larger than 1024 bytes (less than or equal to 512KB), which are sorted in ascending order from smallest to largest
  - Fast one-way linked table (Lookaside) (fast table)
    - The fast table contains 128 entries and is organized similarly to the empty table, but the single-linked table
    - Always initialized to null, each fast table has up to 4 nodes
    - Each node is initialized as occupied, so no heap block merging occurs

#### Management Policy

- Heap Block Allocation
  - Zero empty table allocation: chaining free blocks of different sizes in ascending order, looking for the last block in reverse from `free[0]`, and then searching forward for the smallest free heap block that can meet the requirements for allocation
  - Ordinary free table allocation: find the best free space allocation, followed by the next best
  - Fast table allocation: find the table with matching size, offload it from the heap table, and return a pointer to the heap block to the program
  - When the empty table cannot find the optimal heap block, a slightly larger block is used for allocation. This is a suboptimal allocation, where a block is first cut out from the larger block to the exact size requested for allocation, and then the remaining part is relabeled with the block header and concatenated into the empty table.
  - The fast table is only allocated when it is an exact match, so there is no such phenomenon
- Heap block release
  - Change the heap block status to free and chain to the appropriate heap table. All freed blocks will be chained to the end of the heap table, and the allocation will be taken from the end of the heap table first.
- Heap block merge
  - Repeatedly requesting and releasing heap areas will create a lot of memory fragments, so in order to use memory wisely and efficiently some heap blocks will be merged
  - This operation consists of unloading two blocks from the free table, merging the heap blocks, adjusting the block head information of the merged block, and re-chaining the new block into the free table.
  - The heap area will also undergo a memory crunch (shrink the compact) performed by `RtlCompactHeap`, which will adjust the entire heap and try to merge the available pieces
- Heap block allocation and release strategy
  - Small blocks (SIZE<1KB)
    - Allocation
      - Fast table allocation first, mechanical energy ordinary empty table allocation
      - If it fails, use heap cache allocation
      - If the heap cache allocation fails, try to allocate after memory crunch
      - If allocation is not possible, return NULL
    - Release
      - Priority chaining to fast table (only 4 free blocks can be chained)
      - If the fast table is full, chain to the corresponding empty table
  - Large blocks (1KB<=SIZE<512KB)
    - Allocation
      - Allocate using heap cache
      - If the heap cache allocation fails, use the big block in free[0] for allocation
    - Release
      - Put it into heap cache first
      - If heap cache is full, chain into freelists[0]
  - Jumbo block (SIZE>=512KB)
    - Allocation: dummy allocation (not from heap area)
    - Release: direct release, no heap table operation

---

## Practice

> Lesson learned in blood: whether empty or fast table, its `Blink/Flink` pointer points to **always** the `Blink/Flink` of the next/previous node

### Testing empty tables

#### code

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

#### Watch

##### INT exception calls up debugger, not running

- After `HeadCreate()` creates the heap area, give the heap area pointer to EAX, and observe that the address is `0x360000` at this point
- Check the memory area `0x360000`, the information backward is (copied, I don't know how big these structures are anyway) segment table index (SegmentList), virtual table index (VirtualAllocationList), empty table usage mark (freelist usage bitmap) and empty table index area
- The empty table index is found at offset `0x178`, and its content is `0x00360688`, which means `freelist[0]` points to the offset `0x688`, let's congratulate what is stored in this place
- This place stores `0x00360178`, wonderful, it points to `freelist[0]`, it goes around and points to itself, and this `freelist[0]` seems to point to the only free heap area, generally known as the "tail block"
- According to the structure of the heap block (below), the actual block starts at `0x00360680`, and it looks like the heap block pointer crosses the heap block head and points directly to the data area
- `0x1-0x2` bytes is its own size, at this time the value is `0x0130`, indicating that the size of the heap is `0x130` bytes
- The `0x3-0x4` bytes are the previous heap block size, this value is `0x08` (?????). Isn't it said to be unique???)
- The `0x5` byte is the index, which is 0 at this point
- `0x6` byte is Flag, at this point this is 1
- `0x7` byte is reserved byte, which is 0
- `0x8` byte is the tag index (debug state), I don't know what it does, it is 0
- `0x9-0xC` (empty block exclusive) byte is the address of the previous empty block, it is `0x00360178`
- The `0xD-0x10` (empty heap block exclusive) byte is the address of the next empty heap block, again `0x00360178`

##### runs six allocations

- `0x00360680-0x00360688` is the h1 block header, `0x00360689-0x0036068F` is the 8 byte block body with `00 00 00 00 78 01 36 00`
- `0x00360690-0x00360698` is the h2 block header, `0x00360699-0x0036069F` is the 8-byte block body with `00 00 00 00 00 01 36 00`
- `0x003606A0-0x003606A8` is the h3 block header, `0x003606A9-0x003606AF` is the 8-byte block body with `00 00 00 00 00 00 00 36 00`
- `0x003606B0-0x003606B8` is the h4 block header, `0x003606B9-0x003606BF` is the 8-byte block body with `00 00 00 00 00 00 00 00 00`
- `0x003606C0-0x003606C8` is the h5 block header, `0x003606C9-0x003606DF` is a 24 byte block body with `00 00 00 00 00 00 00 00 00 00 00
- `0x003606E0-0x003606E8` is the h5 block header and `0x003606E9-0x003606FF` is the 24 byte block body with `00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00`

##### Freeing the heap

- The first time the heap is freed the block size is 16 bytes, so it is connected to `freeList[2]`, which is the location of `0x188`, at this time the content is `0x00360688`
- The second freed heap is also connected to `freeList[2]`, which is not described in detail
- The third released heap is also connected to `freeList[2]`, which is not described in detail
- The fourth time it is freed, h3, h4 and h5 are adjacent to each other, so they are merged, where h3h4 is 2 heap units each and h5 is 4, so they are merged to a total of 8 heap units, excluding the heap head, they are left with 7 heap units, so they are put into `freeList[8]`

#### Conclusion

- The information contained in the heap table is SegmentList, VirtualAllocationList, freelist usage bitmap and empty index area in that order.
- When a heap is just initialized, its heap block status
  - There is only one large block in the idle state, which is called the "tail block"
  - This is followed by the fast table
  - Freelist[0] points to the "tail block"
  - Each index of the region points to itself, except for the zero freelist index, which means that there are no free blocks in all the rest of the free-table
- First occupied block

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

- Idle state block head

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

- Heap block allocation
  - The size of the heap block includes the block header, so the application for 32 bytes will allocate 40 bytes.
  - The unit of heap block is 8 bytes, less than 8 bytes are allocated according to 8 bytes, so the minimum actual allocation is 16 bytes
  - In the initial state, the fast table and the empty table are empty, there is no exact allocation, the request will be allocated using the suboptimal block
  - Due to the occurrence of suboptimal allocation, the allocation function will cut away some small blocks from the tail block, modify the size at the beginning of the tail block, and finally point `freelist[0]` to the new tail block

### Test the fast table

#### code

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

#### Conclusion

- The block first identification bit is 0x01
- Only the pointer to the next block in the stack is stored, there is no pointer to the previous block in the stack
- The address of `freeList[0]` at offset `0x178` becomes `0x00361E90` and the original `0x00360688` is occupied by the fast table
- The fast table starts at `0x688`, each structure has a total of `0x30` bytes, and the first four bytes of content are the fast table chain
- Although the 0Day security book says that the 8-byte heap area is inserted as `lookaside[1]`, it seems to me that it is the one at `0x688` that is `lookaside[0]`, the one at `0x6B8` that is `lookaside[1]`, and the one at `0x0E8` that can be called `lookaside[2]`, which The size of the heap block with the block header is 16 bytes in total
