# LAB1 EX3
##Q:分析bootloader进入保护模式的过程。（要求在报告中写出分析）
BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。

提示：需要阅读小节“保护模式和分段机制”和lab1/boot/bootasm.S源码，了解如何从实模式切换到保护模式，需要了解：

    为何开启A20，以及如何开启A20
    如何初始化GDT表
    如何使能和进入保护模式
### Q1：

```
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60

```
以上代码是开启A20的代码，根据实验指导书，A20不开启只能寻址20位的最大地址，为了进入32位的保护模式，需要开启A20,根据实验指导书，
    1等待8042 Input buffer为空；
    2发送Write 8042 Output Port （P2）命令到8042 Input buffer；
    3等待8042 Input buffer为空；
    4将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer；
大概需要4步，按理首先是需要禁止键盘操作的，这里个人认为是汇编的第一句就禁用了中断？

#Q2:
```
lgdt gdtdesc

```
首先这句话将从gdtdesc读48位到GDTR这个寄存器中，这个寄存器的32位基地址保存GDT的地址，16位表示段界限：规定段的大小。由于GDT 不能有GDT本身之内的描述符进行描述定义，所以处理器采用GDTR为GDT这一特殊的系统段。注意，全部描述符表中第一个段描述符设定为空段描述符。GDTR中的段界限以字节为单位。对于含有N个描述符的描述符表的段界限通常可设为8*N-1。所以代码里这个被设置为0×17即23（包含三个段描述符）

```
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt

```

```
#define SEG_NULLASM                                             \
    .word 0, 0;                                                 \
    .byte 0, 0, 0, 0

#define SEG_ASM(type,base,lim)                                  \
    .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
    .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
        (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```
这里.word应该是两个字节16位,.long则是4个字节32位（保存GDT的地址），（应该是还没进入32位的保护模式）头文件里的两个宏定义则描述了一个空的段描述符，和暂时没看懂得可能是如何插入并找到一个段描述符的方式，这里根据宏定义，应该是根据段的类型，基地址，和段的长度（这里固定为0和4G?）来定义的


> 段描述符
在分段存储管理机制的保护模式下，每个段由如下三个参数进行定义：段基地址(Base Address)、段界限(Limit)和段属性(Attributes)。在ucore中的kern/mm/mmu.h中的struct segdesc 数据结构中有具体的定义。
	段基地址：规定线性地址空间中段的起始地址。在80386保护模式下，段基地址长32位。因为基地址长度与寻址地址的长度相同，所以任何一个段都可以从32位线性地址空间中的任何一个字节开始，而不象实方式下规定的边界必须被16整除。
    段界限：规定段的大小。在80386保护模式下，段界限用20位表示，而且段界限可以是以字节为单位或以4K字节为单位。
    段属性：确定段的各种性质。
        段属性中的粒度位（Granularity），用符号G标记。G=0表示段界限以字节位位单位，20位的界限可表示的范围是1字节至1M字节，增量为1字节；G=1表示段界限以4K字节为单位，于是20位的界限可表示的范围是4K
        段属性中的粒度位（Granularity），用符号G标记。G=0表示段界限以字节位位单位，20位的界限可表示的范围是1字节至1M字节，增量为1字节；G=1表示段界限以4K字节为单位，于是20位的界限可表示的范围是4K字节至4G字节，增量为4K字节。
        类型（TYPE）：用于区别不同类型的描述符。可表示所描述的段是代码段还是数据段，所描述的段是否可读/写/执行，段的扩展方向等。
        描述符特权级（Descriptor Privilege Level）（DPL）：用来实现保护机制。
        段存在位（Segment-Present bit）：如果这一位为0，则此描述符为非法的，不能被用来实现地址转换。如果一个非法描述符被加载进一个段寄存器，处理器会立即产生异常。图5-4显示了当存在位为0时，描述符的格式。操作系统可以任意的使用被标识为可用（AVAILABLE）的位。
        已访问位（Accessed bit）：当处理器访问该段（当一个指向该段描述符的选择子被加载进一个段寄存器）时，将自动设置访问位。操作系统可清除该位。

##Q3:

```
.set CR0_PE_ON,             0x1                     # protected mode enable flag

```
```
movl %cr0, %eax
orl $CR0_PE_ON, %eax
movl %eax, %cr0

```
如上对cro寄存器和指定的flag相或对最后一位置1开启保护模式


