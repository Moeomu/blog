---
title: Windows Kernel Debugging Learning Notes-002-Structures
description: Windows kernel related learning notes, this one mainly describes the structure of EPROCESS, PEB
date: 2020-10-17 20:23:00+0800
categories:
    - Kernel
tags:
    - Windows
    - Kernel
---

Source: [Moeomu's Blog](/posts/windows-kernel-debugging-learning-notes-002-structures/)

## Non-public kernel structures

Windows has a lot of non-public structures, and some of them are semi-public, and although they have field names, their purpose can only be inferred  
WinDbg can load some kernel debugging symbols, and in these PDB files there is information about some semi-public structures

---

## EPROCESS(KPEB)(Kernel Process Environment Block)

Each process is represented by an EPROCESS structure, which is linked by a two-way chain table

### Structure information

- 0x000 offset is the address of the PCB (Process Control Block), which is located in R0
- 0x0b4 offset is the PID, which is the unique identifier that identifies this process
- 0x0b8 offset is the active process chain table, which can be used to traverse all EPROCESS structures of the system

### Structure composition

> Here are the details of this structure

```x86asm
kd> dt _eprocess
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS // 进程控制块
   +0x098 ProcessLock      : _EX_PUSH_LOCK // 进程锁
   +0x0a0 CreateTime       : _LARGE_INTEGER // 创建时间
   +0x0a8 ExitTime         : _LARGE_INTEGER // 退出时间
   +0x0b0 RundownProtect   : _EX_RUNDOWN_REF // 进程加保护
   +0x0b4 UniqueProcessId  : Ptr32 Void // PID
   +0x0b8 ActiveProcessLinks : _LIST_ENTRY // 活动进程链表
   +0x0c0 ProcessQuotaUsage : [2] Uint4B // 物理页相关的统计信息
   +0x0c8 ProcessQuotaPeak : [2] Uint4B // 物理页相关的统计信息
   +0x0d0 CommitCharge     : Uint4B
   +0x0d4 QuotaBlock       : Ptr32 _EPROCESS_QUOTA_BLOCK
   +0x0d8 CpuQuotaBlock    : Ptr32 _PS_CPU_QUOTA_BLOCK
   +0x0dc PeakVirtualSize  : Uint4B // 虚拟内存池大小
   +0x0e0 VirtualSize      : Uint4B // 虚拟内存大小
   +0x0e4 SessionProcessLinks : _LIST_ENTRY
   +0x0ec DebugPort        : Ptr32 Void // 调试端口
   +0x0f0 ExceptionPortData : Ptr32 Void
   +0x0f0 ExceptionPortValue : Uint4B
   +0x0f0 ExceptionPortState : Pos 0, 3 Bits
   +0x0f4 ObjectTable      : Ptr32 _HANDLE_TABLE
   +0x0f8 Token            : _EX_FAST_REF // 权限令牌的地址
   +0x0fc WorkingSetPage   : Uint4B
   +0x100 AddressCreationLock : _EX_PUSH_LOCK
   +0x104 RotateInProgress : Ptr32 _ETHREAD
   +0x108 ForkInProgress   : Ptr32 _ETHREAD
   +0x10c HardwareTrigger  : Uint4B
   +0x110 PhysicalVadRoot  : Ptr32 _MM_AVL_TABLE
   +0x114 CloneRoot        : Ptr32 Void
   +0x118 NumberOfPrivatePages : Uint4B
   +0x11c NumberOfLockedPages : Uint4B
   +0x120 Win32Process     : Ptr32 Void
   +0x124 Job              : Ptr32 _EJOB
   +0x128 SectionObject    : Ptr32 Void
   +0x12c SectionBaseAddress : Ptr32 Void
   +0x130 Cookie           : Uint4B
   +0x134 Spare8           : Uint4B
   +0x138 WorkingSetWatch  : Ptr32 _PAGEFAULT_HISTORY
   +0x13c Win32WindowStation : Ptr32 Void
   +0x140 InheritedFromUniqueProcessId : Ptr32 Void
   +0x144 LdtInformation   : Ptr32 Void
   +0x148 VdmObjects       : Ptr32 Void
   +0x14c ConsoleHostProcess : Uint4B
   +0x150 DeviceMap        : Ptr32 Void
   +0x154 EtwDataSource    : Ptr32 Void
   +0x158 FreeTebHint      : Ptr32 Void
   +0x160 PageDirectoryPte : _HARDWARE_PTE
   +0x160 Filler           : Uint8B
   +0x168 Session          : Ptr32 Void
   +0x16c ImageFileName    : [15] UChar
   +0x17b PriorityClass    : UChar
   +0x17c JobLinks         : _LIST_ENTRY
   +0x184 LockedPagesList  : Ptr32 Void
   +0x188 ThreadListHead   : _LIST_ENTRY // ETHREAD结构链表头
   +0x190 SecurityPort     : Ptr32 Void
   +0x194 PaeTop           : Ptr32 Void
   +0x198 ActiveThreads    : Uint4B // 正在运行的线程数量
   +0x19c ImagePathHash    : Uint4B
   +0x1a0 DefaultHardErrorProcessing : Uint4B
   +0x1a4 LastThreadExitStatus : Int4B
   +0x1a8 Peb              : Ptr32 _PEB // 进程环境块地址
   +0x1ac PrefetchTrace    : _EX_FAST_REF
   +0x1b0 ReadOperationCount : _LARGE_INTEGER
   +0x1b8 WriteOperationCount : _LARGE_INTEGER
   +0x1c0 OtherOperationCount : _LARGE_INTEGER
   +0x1c8 ReadTransferCount : _LARGE_INTEGER
   +0x1d0 WriteTransferCount : _LARGE_INTEGER
   +0x1d8 OtherTransferCount : _LARGE_INTEGER
   +0x1e0 CommitChargeLimit : Uint4B
   +0x1e4 CommitChargePeak : Uint4B
   +0x1e8 AweInfo          : Ptr32 Void
   +0x1ec SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
   +0x1f0 Vm               : _MMSUPPORT
   +0x25c MmProcessLinks   : _LIST_ENTRY
   +0x264 HighestUserAddress : Ptr32 Void
   +0x268 ModifiedPageCount : Uint4B
   +0x26c Flags2           : Uint4B
   +0x26c JobNotReallyActive : Pos 0, 1 Bit
   +0x26c AccountingFolded : Pos 1, 1 Bit
   +0x26c NewProcessReported : Pos 2, 1 Bit
   +0x26c ExitProcessReported : Pos 3, 1 Bit
   +0x26c ReportCommitChanges : Pos 4, 1 Bit
   +0x26c LastReportMemory : Pos 5, 1 Bit
   +0x26c ReportPhysicalPageChanges : Pos 6, 1 Bit
   +0x26c HandleTableRundown : Pos 7, 1 Bit
   +0x26c NeedsHandleRundown : Pos 8, 1 Bit
   +0x26c RefTraceEnabled  : Pos 9, 1 Bit
   +0x26c NumaAware        : Pos 10, 1 Bit
   +0x26c ProtectedProcess : Pos 11, 1 Bit
   +0x26c DefaultPagePriority : Pos 12, 3 Bits
   +0x26c PrimaryTokenFrozen : Pos 15, 1 Bit
   +0x26c ProcessVerifierTarget : Pos 16, 1 Bit
   +0x26c StackRandomizationDisabled : Pos 17, 1 Bit
   +0x26c AffinityPermanent : Pos 18, 1 Bit
   +0x26c AffinityUpdateEnable : Pos 19, 1 Bit
   +0x26c PropagateNode    : Pos 20, 1 Bit
   +0x26c ExplicitAffinity : Pos 21, 1 Bit
   +0x270 Flags            : Uint4B
   +0x270 CreateReported   : Pos 0, 1 Bit
   +0x270 NoDebugInherit   : Pos 1, 1 Bit
   +0x270 ProcessExiting   : Pos 2, 1 Bit
   +0x270 ProcessDelete    : Pos 3, 1 Bit
   +0x270 Wow64SplitPages  : Pos 4, 1 Bit
   +0x270 VmDeleted        : Pos 5, 1 Bit
   +0x270 OutswapEnabled   : Pos 6, 1 Bit
   +0x270 Outswapped       : Pos 7, 1 Bit
   +0x270 ForkFailed       : Pos 8, 1 Bit
   +0x270 Wow64VaSpace4Gb  : Pos 9, 1 Bit
   +0x270 AddressSpaceInitialized : Pos 10, 2 Bits
   +0x270 SetTimerResolution : Pos 12, 1 Bit
   +0x270 BreakOnTermination : Pos 13, 1 Bit
   +0x270 DeprioritizeViews : Pos 14, 1 Bit
   +0x270 WriteWatch       : Pos 15, 1 Bit
   +0x270 ProcessInSession : Pos 16, 1 Bit
   +0x270 OverrideAddressSpace : Pos 17, 1 Bit
   +0x270 HasAddressSpace  : Pos 18, 1 Bit
   +0x270 LaunchPrefetched : Pos 19, 1 Bit
   +0x270 InjectInpageErrors : Pos 20, 1 Bit
   +0x270 VmTopDown        : Pos 21, 1 Bit
   +0x270 ImageNotifyDone  : Pos 22, 1 Bit
   +0x270 PdeUpdateNeeded  : Pos 23, 1 Bit
   +0x270 VdmAllowed       : Pos 24, 1 Bit
   +0x270 CrossSessionCreate : Pos 25, 1 Bit
   +0x270 ProcessInserted  : Pos 26, 1 Bit
   +0x270 DefaultIoPriority : Pos 27, 3 Bits
   +0x270 ProcessSelfDelete : Pos 30, 1 Bit
   +0x270 SetTimerResolutionLink : Pos 31, 1 Bit
   +0x274 ExitStatus       : Int4B
   +0x278 VadRoot          : _MM_AVL_TABLE
   +0x298 AlpcContext      : _ALPC_PROCESS_CONTEXT
   +0x2a8 TimerResolutionLink : _LIST_ENTRY
   +0x2b0 RequestedTimerResolution : Uint4B
   +0x2b4 ActiveThreadsHighWatermark : Uint4B
   +0x2b8 SmallestTimerResolution : Uint4B
   +0x2bc TimerResolutionStackRecord : Ptr32 _PO_DIAG_STACK_RECORD
```

---

## PEB(Process Environment Block)

This structure is located at the R3 level and is relatively easy to modify

### Structure information

- 0x002 offset is the location of the FLAG whether to be debugged or not, this value can be modified under R3
- 0x068 offset is the value of 0 normally, 0x70 when debugged
- 0x018 offset is the address of _HEAP structure, this structure can be judged as non-debug state when offset 0x40=2 and 0x44=0

### Structure composition

```x86asm
kd> dt _PEB
nt!_PEB
   +0x000 InheritedAddressSpace : UChar
   +0x001 ReadImageFileExecOptions : UChar
   +0x002 BeingDebugged    : UChar // 是否被调试
   +0x003 BitField         : UChar
   +0x003 ImageUsesLargePages : Pos 0, 1 Bit
   +0x003 IsProtectedProcess : Pos 1, 1 Bit
   +0x003 IsLegacyProcess  : Pos 2, 1 Bit
   +0x003 IsImageDynamicallyRelocated : Pos 3, 1 Bit
   +0x003 SkipPatchingUser32Forwarders : Pos 4, 1 Bit
   +0x003 SpareBits        : Pos 5, 3 Bits
   +0x004 Mutant           : Ptr32 Void
   +0x008 ImageBaseAddress : Ptr32 Void
   +0x00c Ldr              : Ptr32 _PEB_LDR_DATA // 进程装载的模块结构体
   +0x010 ProcessParameters : Ptr32 _RTL_USER_PROCESS_PARAMETERS
   +0x014 SubSystemData    : Ptr32 Void
   +0x018 ProcessHeap      : _HEAP // 0x40=2&&0x44=0为非调试状态
   +0x01c FastPebLock      : Ptr32 _RTL_CRITICAL_SECTION
   +0x020 AtlThunkSListPtr : Ptr32 Void
   +0x024 IFEOKey          : Ptr32 Void
   +0x028 CrossProcessFlags : Uint4B
   +0x028 ProcessInJob     : Pos 0, 1 Bit
   +0x028 ProcessInitializing : Pos 1, 1 Bit
   +0x028 ProcessUsingVEH  : Pos 2, 1 Bit
   +0x028 ProcessUsingVCH  : Pos 3, 1 Bit
   +0x028 ProcessUsingFTH  : Pos 4, 1 Bit
   +0x028 ReservedBits0    : Pos 5, 27 Bits
   +0x02c KernelCallbackTable : Ptr32 Void
   +0x02c UserSharedInfoPtr : Ptr32 Void
   +0x030 SystemReserved   : [1] Uint4B
   +0x034 AtlThunkSListPtr32 : Uint4B
   +0x038 ApiSetMap        : Ptr32 Void
   +0x03c TlsExpansionCounter : Uint4B
   +0x040 TlsBitmap        : Ptr32 Void
   +0x044 TlsBitmapBits    : [2] Uint4B
   +0x04c ReadOnlySharedMemoryBase : Ptr32 Void
   +0x050 HotpatchInformation : Ptr32 Void
   +0x054 ReadOnlyStaticServerData : Ptr32 Ptr32 Void
   +0x058 AnsiCodePageData : Ptr32 Void
   +0x05c OemCodePageData  : Ptr32 Void
   +0x060 UnicodeCaseTableData : Ptr32 Void
   +0x064 NumberOfProcessors : Uint4B
   +0x068 NtGlobalFlag     : Uint4B // 反调试用
   +0x070 CriticalSectionTimeout : _LARGE_INTEGER
   +0x078 HeapSegmentReserve : Uint4B
   +0x07c HeapSegmentCommit : Uint4B
   +0x080 HeapDeCommitTotalFreeThreshold : Uint4B
   +0x084 HeapDeCommitFreeBlockThreshold : Uint4B
   +0x088 NumberOfHeaps    : Uint4B
   +0x08c MaximumNumberOfHeaps : Uint4B
   +0x090 ProcessHeaps     : Ptr32 Ptr32 Void
   +0x094 GdiSharedHandleTable : Ptr32 Void
   +0x098 ProcessStarterHelper : Ptr32 Void
   +0x09c GdiDCAttributeList : Uint4B
   +0x0a0 LoaderLock       : Ptr32 _RTL_CRITICAL_SECTION
   +0x0a4 OSMajorVersion   : Uint4B // 系统主版本号
   +0x0a8 OSMinorVersion   : Uint4B // 系统子版本号
   +0x0ac OSBuildNumber    : Uint2B // 系统构建版本号
   +0x0ae OSCSDVersion     : Uint2B
   +0x0b0 OSPlatformId     : Uint4B
   +0x0b4 ImageSubsystem   : Uint4B
   +0x0b8 ImageSubsystemMajorVersion : Uint4B
   +0x0bc ImageSubsystemMinorVersion : Uint4B
   +0x0c0 ActiveProcessAffinityMask : Uint4B
   +0x0c4 GdiHandleBuffer  : [34] Uint4B
   +0x14c PostProcessInitRoutine : Ptr32 void
   +0x150 TlsExpansionBitmap : Ptr32 Void
   +0x154 TlsExpansionBitmapBits : [32] Uint4B
   +0x1d4 SessionId        : Uint4B
   +0x1d8 AppCompatFlags   : _ULARGE_INTEGER
   +0x1e0 AppCompatFlagsUser : _ULARGE_INTEGER
   +0x1e8 pShimData        : Ptr32 Void
   +0x1ec AppCompatInfo    : Ptr32 Void
   +0x1f0 CSDVersion       : _UNICODE_STRING
   +0x1f8 ActivationContextData : Ptr32 _ACTIVATION_CONTEXT_DATA
   +0x1fc ProcessAssemblyStorageMap : Ptr32 _ASSEMBLY_STORAGE_MAP
   +0x200 SystemDefaultActivationContextData : Ptr32 _ACTIVATION_CONTEXT_DATA
   +0x204 SystemAssemblyStorageMap : Ptr32 _ASSEMBLY_STORAGE_MAP
   +0x208 MinimumStackCommit : Uint4B
   +0x20c FlsCallback      : Ptr32 _FLS_CALLBACK_INFO
   +0x210 FlsListHead      : _LIST_ENTRY
   +0x218 FlsBitmap        : Ptr32 Void
   +0x21c FlsBitmapBits    : [4] Uint4B
   +0x22c FlsHighIndex     : Uint4B
   +0x230 WerRegistrationData : Ptr32 Void
   +0x234 WerShipAssertPtr : Ptr32 Void
   +0x238 pContextData     : Ptr32 Void
   +0x23c pImageHeaderHash : Ptr32 Void
   +0x240 TracingFlags     : Uint4B
   +0x240 HeapTracingEnabled : Pos 0, 1 Bit
   +0x240 CritSecTracingEnabled : Pos 1, 1 Bit
   +0x240 SpareTracingBits : Pos 2, 30 Bits

```

---

## Reference Documentation

[1]Infosavvy.Understanding EProcess Structure[J/OL].2020-07-24
