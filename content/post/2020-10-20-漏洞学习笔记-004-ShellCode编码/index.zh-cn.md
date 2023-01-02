---
title: 漏洞利用学习笔记-004-ShellCode编码
description: ShellCode编码和减少体积
date: 2020-10-20 9:20:00+0800
categories:
    - 漏洞利用
tags:
    - Windows
    - ShellCode
---

> [点击此处下载本文附可执行程序，shellcode文件](exploit-study-04.zip)

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞利用学习笔记-004-shellcode编码/)

## 异或编码

### 注意事项

- 在选取编码字节时，不可与已有字节相同，否则会出现0
- 可以使用多个不同编码字节对不同区域编码，但会增加复杂度
- 可以对shellcode进行多轮编码

### 实现代码(ExpStd0401)

```CPP
void encoder(char* input, unsigned char key, int display_flag)
{
    int i = 0, len = 0;
    FILE* fp;
    unsigned char * output;
    len = strlen(input);
    output = (unsigned char*)malloc(len + 1);
    if(!output)
    {
        printf("memory error!\n");
        exit(0);
    }

    // encode shellcode
    for(i = 0; i < len; i++)
    {
        output[i] = input[i] ^ key;
    }
    if(!(fp=fopen("encode.txt", "w+")))
    {
        printf("output file create error!");
        exit(0);
    }
    fprintf(fp, "\"");
    for(i = 0; i< len; i++)
    {
        fprintf(fp, "\\x%0.2x", output[i]);
        if((i + 1 % 16 == 0))
        {
            fprintf(fp, "\"\n\"");
        }
    }
    fprintf(fp, "\";");
    fclose(fp);
    printf("dump the encoded shellcode to encode.txt OK!\n");
    if(display_flag)
    {
        for(i = 0; i < len; i++)
        {
            printf("%0.2x ", output[i]);
            if((i + 1) % 16 == 0)
            {
                printf("\n");
            }
        }
    }
    free(output);
}
```

### 解码代码(ExpStd0402)

- 解码器与shellcode联合执行
- 默认EAX在shellcode开始时对准shellcode起始位置
- shellcode最后一个字节为0x90

```x86asm
void main()
{
    __asm
    {
        add eax, 0x14            ;越过decoder记录shellcode起始地址
        xor ecx, ecx
    decode_loop:
        mov bl, [eax + ecx]
        xor bl, 0x44             ;用0x44作为key
        mov [eax + ecx], bl
        inc ecx
        cmp bl, 0x90             ;末尾放一个0x90作为结束符
        jne decode_loop
    }
}
```

## ShellCode减少体积

### 方法

- 挑选短指令
  - `xchg eax, reg`   ;交换`eax`和其它寄存器的值
  - `lodsb`           ;`esi`指向的一个`dword`装入`eax`，并且增加`esi`
  - `lodsd`           ;把`esi`指向的一个`byte`装入`al`，并增加`esi`
  - `stosd`           ;将`eax`的内容复制到`edi`的内存地址中，每复制四个字节，`edi`就加`0x4`，`ecx`为大小
  - `stosb`           ;将`eax`的内容复制到`edi`的内存地址中，每复制一个字节，`edi`就加`0x4`，`ecx`为大小
  - `pushad/popad`    ;从栈中存储/恢复所有寄存器的值
  - `cdq`             ;用`edx`把`eax`扩展成四字，在`eax<0x80000000`时可用作`mov edx, 0`
- 复合指令，合并使用指令
- API参数压栈前将栈空间一片区域置为0，压栈时只要压入非0参数即可
- 代码当数据用，数据当代码用
- 栈顶之上数据若有用，抬高栈顶保护它以便以后使用
- 调用API时有些寄存器总是被保存在栈中，但是大多数函数运行时不会使用EBP，因此可以用EBP保存数据
- HASH算法存储API

### 选择适当的HASH算法

- 8bit最多表示256个不同的字符，不可避免会有碰撞，但是如果所需函数位于碰撞的第一个，那么也可以使用
- 即碰撞是可以部分容忍的
