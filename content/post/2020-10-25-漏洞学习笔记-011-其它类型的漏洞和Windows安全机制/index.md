---
title: Exploit learning notes 011 other types of exploits and Windows Security Mechanisms
description: Overview of other types of vulnerabilities and Windows security mechanisms
date: 2020-10-25 18:50:00+0800
categories:
    - Exploit
tags:
    - Windows
    - SecurityMechanisms
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-011-other-types-of-exploits-and-windows-security-mechanisms/)

## Formatting string vulnerability

### Flaw in printf

> Example

```CPP
#include "stdio.h"

void main()
{
    int a = 44,b = 77;
    printf("a=%d, b=%d\n",a,b);
    printf("a=%d, b=%d\n");
}
```

- The second call in the above code is missing the list of variables for the output data
- However, the second call does not cause a compilation error and the program executes normally

### Reading memory data with printf

> Example

```CPP
#include "stdio.h"

int main(int argc, char ** argv)
{
    printf(argv[1]);
}
```

- When we pass a normal string to the program, we get a normal string
- But if it comes with a format control character, you can read the data on the stack

### Write data to memory with printf

> Example

```CPP
#include "stdio.h"
int main(int argc, char ** argv)
{
    int len_print = 0;
    printf("before write: length=%d\n", len_print);
    printf("Misaka:%d%n\n",len_print, &len_print);
    printf("after write: length=%d\n", len_print);
}
```

- The %n controller calculates the length of the output string and then writes it back to the len_print variable

## SQL injection attack

### Principle

- It stems from a flaw in PHP, ASP and other scripting languages when heaping user input data and parsing it

### It is not a binary vulnerability and will not be discussed here

### Windows security mechanism

### Flaw in Turing machine

- There is no clear distinction between code and data, so there are always problems
- For example, stack overflow attacks, shelling and deshelling techniques, morphing virus techniques
- Cross-site scripting attacks, SQL injection attacks are also caused by this flaw

### Changes in Windows

#### Macro changes

- Windows Security Center was added
- Added firewall to Windows
- Web pop-ups and ActiveX control installation will be disabled without permission
- IE7 added a feature to filter counterfeit websites
- Added UAC (User Account Control) mechanism to prevent malicious software from being installed on the computer without permission or from making changes to the computer
- Integrated Windows Defender to block, control and remove spyware and malware

#### Memory security changes

- Added `SecurityCookie` before the function return address using GS compilation technology, which first detects whether `SecurityCookie` is overwritten before the function returns and stack overflow becomes difficult
- Added the security check mechanism of heap SEH to effectively prevent most of the attacks of rewriting SEH and hijacking the process.
- Heap security mechanisms such as `Heap Cookie` and `Safe Unlinking` have been added to the heap, and the heap overflow is more restricted.
- `DEP(Data Execution Protection)` Data Execution Protection marks data sections as non-executable, preventing the execution of attack code in the stack, heap and data sections
- `ASLR(Address Space Layout Randomization)`Load address randomization technique disables classical stack overflow techniques by randomizing key addresses in the heap system
- `SEHOP(Structured Exception Handler Overwrite Protection)`SEH overwrite protection complements the heap SEH security mechanism by raising the SEH protection to the system level, making the SEH protection mechanism more effective

#### Summary of Windows security mechanisms

|  | Windows XP | Windows 2003 | Windows Vista | Windows 2008 | Windows 7 |
| - | - | - | - | - | - |
| **GS** |   |   |   |   |   |
| Security Cookies | √ | √ | √ | √ | √ |
| Variable rearrangement | √ | √ | √ | √ | √ |
| **Safety SEH** |   |   |   |   |   |
| SEH handle validation | √ | √ | √ | √ | √ |
| **Heap Protection** |   |   |   |   |   |
| Safe disassembly | √ | √ | √ | √ | √ |
| Safety Fast Meter | × | × | √ | √ | √ |
| Heap Cookie | √ | √ | √ | √ | √ |
| Metadata Encryption | × | × | √ | √ | √ |
| **DEP** |   |   |   |   |   |
| NX Support | √ | √ | √ | √ | √ |
| Permanent DEP | × | × | √ | √ | √ |
| Default OptOut | × | √ | × | √ | × |
| **ASLR** |   |   |   |   |   |
| PEB, TEB | √ | √ | √ | √ | √ |
| Heap | × | × | √ | √ | √ |
| Stack | × | × | √ | √ | √ |
| Image | × | × | √ | √ | √ |
| **SEHOP** |   |   |   |   |   |
 SEH chain validation | × | × | √ | √ | √ |
