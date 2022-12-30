# LAB1 EX4
## 分析bootloader加载ELF格式的OS的过程。
	通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试bootloader&OS，

    bootloader如何读取硬盘扇区的？
    bootloader是如何加载ELF格式的OS？
###Q1:
根据小节“硬盘访问概述”

	当前 硬盘数据是储存到硬盘扇区中，一个扇区大小为512字节。读一个扇区的流程（可参看boot/bootmain.c中的readsect函数实现）大致如下：

    等待磁盘准备好
    发出读取扇区的命令
    等待磁盘准备好
    把磁盘扇区数据读到指定内存
bootmain.c中的readsect是读取一个指定的扇区至内存（512bytes）
readsec这个函数是从可执行文件kernel的offset处读取count个字节，但由于是基于readsect这个函数实现的，所以看以下代码
```
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```
这里代码中va减去offset处以512的余数是为了使得输入的参数va对准的是kernel取offset后的位置

###Q2:
在bootmain函数中加载了elf格式的OS
```
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```
首先读取kernel到内存中，检查是否为elf格式，然后根据elfheader中的e_phoff参数地参数定位到programheader,其中phnum参数代表有多少个这样的programheader,根据这些数量和programheader的va(要读到哪个虚拟地址),将这些，memsz(段在内存映像中占用的字节数),offset(段相对文件头的偏移值)依次读取这些段