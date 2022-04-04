---
title: Exploit Learning Notes 004 ShellCode Coding
description: ShellCode encoding and volume reduction
date: 2020-10-20 9:20:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

> [Click here to download this article with executable program, shellcode file](./exploit-study-04.zip)

Source: [Moeomu's blog](/posts/exploit-learning-notes-004-shellcode-coding/)

## Variable code

### Caution

- When picking encoding byte, it can't be the same as existing byte, otherwise there will be 0
- It is possible to encode different areas with multiple different encoding bytes, but it will increase the complexity
- Multiple rounds of encoding of shellcode are possible

### Implementation code (ExpStd0401)

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

### Decoding code (ExpStd0402)

- Decoder is executed jointly with shellcode
- Default EAX is aligned to the shellcode start position at the beginning of the shellcode
- The last byte of shellcode is 0x90

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

## ShellCode to reduce the size

### Methods

- Pick short instruction
  - `xchg eax, reg` ;swap the values of `eax` and other registers
  - `lodsb` ;load a `dword` pointed to by `esi` into `eax` and add `esi`
  - `lodsd` ;load a `byte` pointed to by `esi` into `al` and increment `esi`
  - `stosd` ;copy the contents of `eax` to the memory address of `edi`, adding `0x4` to `edi` for every four bytes copied, and `ecx` for the size
  - `stosb` ; copy the content of `eax` to the memory address of `edi`, for every byte copied, `edi` adds `0x4`, `ecx` is the size
  - `pushad/popad` ;store/restore all register values from the stack
  - `cdq` ;use `edx` to expand `eax` into four words, can be used as `mov edx, 0` when `eax<0x80000000`
- Compound instructions, combined use instructions
- API parameter stacking before a piece of the stack space to 0, the stack can be pressed into the non-0 parameters
- Code is used as data, data is used as code
- If the data on top of the stack is useful, raise the top of the stack to protect it for later use
- Some registers are always stored on the stack when the API is called, but most functions do not use EBP when they run, so you can use EBP to store data.
- HASH algorithm for storing APIs

### Select the appropriate HASH algorithm

- 8bit represents up to 256 different characters, there will inevitably be collisions, but if the desired function is located first in the collision, then it can be used
- i.e. collisions are partially tolerable
