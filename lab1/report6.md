# LAB1 EX6
###完善中断初始化和处理 
##Q:请完成编码工作和回答如下问题：

    中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
    请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。
    请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

##Q1:
8字节，如下图所示，其中的选择址（索引）找到段描述符表中的段描述符,然后找到该段的基地址（这里每一段都是0），再加上offset找到程序入口，ps:这里的GDT和前面EX3里的应该不是一张表了,这是操作系统建立的GDT。
![](../图片/image008.png)
##Q2:
踩了几个坑，

	1、怎么直接从c里拿到汇编定义在全局.data段的数据
首先看vector.S代码：
```
.text
.globl __alltraps
.globl vector0
vector0:
  pushl $0
  pushl $0
  jmp __alltraps
.globl vector1
vector1:
  pushl $0
  pushl $1
  jmp __alltraps
 ...

.data
.globl __vectors
__vectors:
  .long vector0
  .long vector1
  .long vector2
```
这里最下面的__vectors:开始，.long 后面加一个表达式？存的是上面.text段里该表达式的地址，在c语言中想获取这个地址的数组，使用
```
extern uintptr_t __vectors[];
```
即可。

	2、SETGATE宏函数
首先是我的代码
```
//	int i;
//	extern uintptr_t __vectors[];//all ISR ADDRESS
//	struct gatedesc tmp_gate;
//
//	for(i=0;i<256;i++){
//		if(i<32){SETGATE(tmp_gate,1,2,__vectors[i],0)}
//
//			//32 0? behind 1,last dpl i guess3//2 supposed
//			//to be kernel  data segement index
//		else{SETGATE(tmp_gate,0,2,__vectors[i],0)}
//
//		idt[i] = tmp_gate;
//}
//	lidt(&idt_pd);//wait to decide

```
再贴上答案的代码：
```
extern uintptr_t __vectors[];
	    int i;
	    for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
	        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
	    }
		// set for switch from user to kernel
	    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
		// load the IDT
	    lidt(&idt_pd);
```
主要就是SETGATE里的参数设置的不对，首先看下这个宏定义
```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
第一个就是中断向量表的表项，第二个1就是软中段或者异常，0是硬中断，第三个注释里写着是段选择址，然后是偏置，最后是特权级，0是内核，3是用户。
这里我写sel的时候犯了两个错误，第一，收到vector.S这个文件的.data描述符影响，以为它是在内核数据段里，第二，没有将段的索引乘以8,（如下图），然后这里答案把256个中断全设置成硬中断，这点不是很理解，除了在头文件trap.h中定义的T_SWITCH_TOK的DPL设为3,其他全为0。memlayout.h中也定义了许多要用到的常量。


![](/home/oem/图片/image011.png)
###一点小感悟？
这里写的时候只想着参考实验指导书，但这里的一些细节可能还是要看intel的x86开发者手册才能写出来

	3、lidt
##TIPS
<<左移运算符和>>右移运算符，一个数左移几位相当于该数乘上2的几次方，一个数右移几位相当于该数除以2的几次方。
##PS
扩展有空再做，感觉不是很难