# LAB1 EX1
## Q1 ucore.img是如何生成的？
根据Makefile以及make V=的提示下
+ cc kern/init/init.c
+ cc kern/libs/readline.c
+ cc kern/libs/stdio.c
+ cc kern/debug/kdebug.c
+ cc kern/debug/kmonitor.c
+ cc kern/debug/panic.c
+ cc kern/driver/clock.c
+ cc kern/driver/console.c
+ cc kern/driver/intr.c
+ cc kern/driver/picirq.c
+ cc kern/trap/trap.c
+ cc kern/trap/trapentry.S
+ cc kern/trap/vectors.S
+ cc kern/mm/pmm.c
+ cc libs/printfmt.c
+ cc libs/string.c
+ ld bin/kernel
+ cc boot/bootasm.S
+ cc boot/bootmain.c
+ cc tools/sign.c
+ ld bin/bootblock


用到了这些目录下的源代码，其中到+ ld bin/kernel之前是在编译内核用到的c源文件，ld是把编译好后的.o目标文件链接在一起，生成bin目录下的可执行文件

下面4句则应该是对主引导扇区中的系统启动程序等做了编译和链接

接下来通过下面的dd指令将bin目录下的kern和bootblock写入ucore.img

dd if=/dev/zero of=bin/ucore.img count=10000
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc


这里只是简单通过make V=给出的提示做了分析，对Makefile还不是很了解

## Q2 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
根据tools/sign.c中的源代码，其应不超过512个字节，且最后两个字节为
```
buf[510] = 0x55;
buf[511] = 0xAA;

```
