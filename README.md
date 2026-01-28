# 概述

在 **Intel Haswell** 架构里引入了 **Gather** 特性。它使得CPU可以使用向量索引存储器编址从存储器取非连续的数据元素。这些gather指令引入了一种新的存储器寻址形式，该形式由一个 **基地址寄存器**（仍然是通用目的寄存器）和通过一个 **向量寄存器**（**XMM** 或 **YMM**）所指定的多个索引构成。数据元素大小支持32位与64位，并且数据类型支持浮点型和整型。

我们先回顾一下普通的x86寻址方式：`[<base register> + <index register> * <scale> + <offset>]`

在 **AT&T** 汇编语法的形式下表达为：`<offset>(<base register>, <index register>, <scale>)`

其中，**`<base register>`** 为基地址寄存器；**`<index register>`** 为索引寄存器；**`<scale>`** 为刻度因子，它是一个立即数，并仅支持 0，1，2，4，8 这几个值；**`<offset>`** 表示偏移量，它是一个立即数。

那么下面我们就先来谈谈上面所提到的向量存储器寻址。

<br />

# 向量SIB（VSIB）存储器寻址

在AVX2中，跟在 **ModR/M** 字节后面的 **SIB**（**S** 表示 Scale；**I** 表示 Index；**B** 表示 Base）字节可以支持对一组线性地址的 **VSIB** 存储器寻址。**VSIB** 寻址仅在AVX2指令的子集中支持。**VSIB** 存储器寻址要求32位或64位的有效寻址。在32位模式下，当地址大小属性被重载为16位时，**VSIB** 寻址不被支持。在16位保护模式下，**VSIB** 寻址是被允许的，若地址大小属性被重载为32位的话。此外，**VSIB** 存储器寻址仅伴随 **VEX** 前缀而被支持。

在 **VSIB** 存储器寻址中，**SIB** 字节由以下部分组成：
- 刻度域（位7:6）指定了刻度因子。
- 索引域（位5:3）指定了向量索引寄存器的寄存器编号，该向量存储器中的每个元素都指定了一个索引。
- 基地址域（位2:0）指定了基地址寄存器的编号。

比如：
```nasm
.text
.align 4
.att_syntax

vgatherdpd    %xmm0, 128(%rdi, %xmm2, 4), %xmm3
```
或
```nasm
.text
.align 4
.intel_syntax noprefix

vgatherdpd    xmm3, [rdi + xmm2 * 4 + 128], xmm0
```
上述指令中，基地址寄存器为 **`RDI`**，索引寄存器为 **`XMM2`**，刻度因子是4，偏移量是128。而指令 **`vgatherdpd`** 是将索引寄存器的元素作为双字（即4字节）进行划分，然后乘上刻度因子后加到基地址上。而偏移量则作用于每个基地址元素。

下面我们将提供一个比较完整的示例代码来描述 **`VGATHERDPD`** 指令。

先看汇编指令：
```nasm
.text
.align 4
.att_syntax

#ifdef __APPLE__
.globl _InstTest
_InstTest:
#else
.globl InstTest
InstTest:
#endif

    // 设置索引寄存器的每个元素
    mov     $4, %eax
    // 前一个索引为4
    movd    %eax, %xmm2
    mov     $8, %eax
    // 后一个索引为8
    pinsrd  $1, %eax, %xmm2

    // 将两个double元素的mask全都置1
    mov     $0xffffffffffffffff, %rax
    // xmm0作为mask寄存器
    movq    %rax, %xmm0
    punpcklqdq    %xmm0, %xmm0
    // xmm3作为目的寄存器
    vgatherdpd    %xmm0, 8(%rdi, %xmm2, 2), %xmm3

    ret
```
或
```nasm
.text
.align 4
.intel_syntax noprefix

#ifdef __APPLE__
.globl _InstTest
_InstTest:
#else
.globl InstTest
InstTest:
#endif

    // 设置索引寄存器的每个元素
    mov     eax, 4
    // 前一个索引为4
    movd    xmm2, eax
    mov     eax, 8
    // 后一个索引为8
    pinsrd  xmm2, eax, 1

    // 将两个double元素的mask全都置1
    mov     rax, 0xffffffffffffffff
    // xmm0作为mask寄存器
    movq    xmm0, rax
    punpcklqdq    xmm0, xmm0
    // xmm3作为目的寄存器
    vgatherdpd    xmm3, [rdi + xmm2 * 2 + 8], xmm0

    ret
```

这里，**`RDI`** 寄存器作为第一个输入参数，存放了基地址。

下面是C函数的调用：

```c
#include <stdalign.h>

int main(void)
{    
    extern void InstTest(void *p);
    
    alignas(64) unsigned buffer[] = { 0x01020304U, 0x05060708U, 0x090a0b0cU, 0x10121314U, 0x15161718U, 0x191a1b1cU, 0x20212223U, 0x24252627U };
    
    InstTest(buffer);

    return 0;
}
```

我们通过在 `return 0;` 这条语句设置断点，然后通过 GDB 或 LLDB 调试器可以发现最终目的寄存器 **`XMM3`** 的最后内容为：
> xmm3 = {0x18 0x17 0x16 0x15 0x1c 0x1b 0x1a 0x19 0x23 0x22 0x21 0x20 0x27 0x26 0x25 0x24}

以上代码的编译环境为：macOS 10.9.3, Xcode 5.1, Apple LLVM 5.1。

运行环境为：MacBook Air 2013版，Intel Core i7 4650U, 8GB DDR3。

