#LAB1 EX5
##Q:实现函数调用堆栈跟踪函数

    Q1:了解编译器如何建立函数调用关系的。在完成lab1编译后，查看lab1/obj/bootblock.asm，了解bootloader源码与机器码的语句和地址等的对应关系
    理解调用栈最重要的两点是：栈的结构，EBP寄存器的作用。一个函数调用动作可分解为：零到多个PUSH指令（用于参数入栈），一个CALL指令。CALL指令内部其实还暗含了一个将返回地址（即CALL指令下一条指令的地址）压栈的动作（由硬件完成）。
以bootblock.asm和bootloader源码为例，首先反汇编后的汇编显示此时的程序代码对应的汇编已经固定存储在内存中，每次函数调用前即call之前会把call的函数的参数入栈例如下所示
```
  100014:	89 44 24 08          	mov    %eax,0x8(%esp)
  100018:	c7 44 24 04 00 00 00 	movl   $0x0,0x4(%esp)
  10001f:	00 
  100020:	c7 04 24 16 da 10 00 	movl   $0x10da16,(%esp)
  100027:	e8 54 30 00 00       	call   103080 <memset>

```
这段代码是kern/init/init.c中的入口函数调用函数memset的反汇编代码，要传三个参数，X86平台下，分配函数栈空间是由高地址向低地址分配的，所以栈顶在低地址，栈底在高地址，esp是当前函数的栈顶指针，所以可见参数的入栈顺序为倒着来的，高地址存着最后的参数，然后每个被调用的函数的开头两句都是
```
00103080 <memset>:
 * @n:        number of bytes to be set to the value
 *
 * The memset() function returns @s.
 * */
void *
memset(void *s, char c, size_t n) {
  103080:	55                   	push   %ebp
  103081:	89 e5                	mov    %esp,%ebp
```
```
00007c72 <readsect>:
        /* do nothing */;
}

/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    7c72:	55                   	push   %ebp
    7c73:	89 d1                	mov    %edx,%ecx
    7c75:	89 e5                	mov    %esp,%ebp
```
即
```
pushl   %ebp
movl   %esp , %ebp
```
这样的话这样在程序执行到一个函数的实际指令前，已经有以下数据顺序入栈：参数、返回地址、ebp寄存器。ebp可理解为指向栈底

这两条汇编指令的含义是：首先将ebp寄存器入栈，然后将栈顶指针esp赋值给ebp。“mov ebp esp”这条指令表面上看是用esp覆盖ebp原来的值，其实不然。因为给ebp赋值之前，原ebp值已经被压栈（位于栈顶），而新的ebp又恰恰指向栈顶。此时ebp寄存器就已经处于一个非常重要的地位，该寄存器中存储着栈中的一个地址（原ebp入栈后的栈顶），从该地址为基准，向上（栈底方向）能获取返回地址、参数值，向下（栈顶方向）能获取函数局部变量值，而该地址处又存储着上一层函数调用时的ebp值。
一般而言，ss:[ebp+4]处为返回地址，ss:[ebp+8]处为第一个参数值（最后一个入栈的参数值，此处假设其占用4字节内存），ss:[ebp-4]处为第一个局部变量，ss:[ebp]处为上一层ebp值。由于ebp中的地址处总是“上一层函数调用时的ebp值”，而在每一层函数调用中，都能通过当时的ebp值“向上（栈底方向）”能获取返回地址、参数值，“向下（栈顶方向）”能获取函数局部变量值。如此形成递归，直至到达栈底。这就是函数调用栈。

具体看实验指导书函数堆栈小节和书。

##inline关键字
如果编译器选择内联某个函数，那么，在调用该函数后会直接使用宏展开成该函数的汇编而不是调用call，来节省函数调用的资源，使用于一些简单但高频调用的函数。

##实现函数print_stackframe
```
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
	uint32_t ebp = read_ebp();
	uint32_t eip = read_eip();
	int i;
	for(i=0;i<=STACKFRAME_DEPTH;++i){
//(1)

		cprintf("ebp:0x%x ",ebp);

		cprintf("eip:0x%x ",eip);
		cprintf("args:");


		int j;
		for(j=0;j<4;j++){
			uint32_t temp = *(uint32_t *)(ebp+6+j*2);
			cprintf("0x%x ",temp);
		}
		cprintf("\n");
		//(3.4)
		print_debuginfo(eip-1);
		//3.5
		ebp = *(uint32_t *)ebp;
		eip = *(uint32_t *)(ebp+4);
	}

}
```
主要思路，首先读取ebp和eip

    Instruction Pointer(指令指针寄存器)：EIP的低16位就是8086的IP，它存储的是下一条要执行指令的内存地址，在分段地址转换中，表示指令的段内偏移地址。
当前函数栈中ebp寄存器中的地址所指向的是当前栈的ebp位置，当前栈的ebp里的地址又指向上一个ebp	，当前栈的ebp位置高4字节是eip的地址即当前执行指令的内存地址。

首先用ucore自带的stdio中的cprinf打印出这两个值，然后我的理解是打印该函数的4个参数，那么位置应该在ebp+6开始是最后一个参数,要求是uint32_t 类型，因此读出ebp寄存器中的内容（当前函数栈ebp所在地址），将其值+6并转为指针读出其中的内容。
然后调print_debuginfo(eip-1);
然后出栈，这里的出栈应该不是真的出栈？只是改ebp,eip的值？
因此，新的ebp的地址就是将当前的ebp中保存的值取出来，即将当前ebp转为指针，然后解引用，新的eip也是更新后的ebp+4地址中的内容


##解答一些错误
最开始的实现有些问题，但经过半天的bug查找与修改，还是有一些收获的，首先贴出官方代码和一些关键函数
```
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
    uint32_t ebp = read_ebp(), eip = read_eip();

    int i, j;
    for (i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i ++) {
        cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
        uint32_t *args = (uint32_t *)ebp + 2;
        for (j = 0; j < 4; j ++) {
            cprintf("0x%08x ", args[j]);
        }
        cprintf("\n");
        print_debuginfo(eip - 1);
        eip = ((uint32_t *)ebp)[1];
        ebp = ((uint32_t *)ebp)[0];
    }
}

...

static __noinline uint32_t
read_eip(void) {
    uint32_t eip;
    asm volatile("movl 4(%%ebp), %0" : "=r" (eip));
    return eip;
}

...

static inline uint32_t
read_ebp(void) {
    uint32_t ebp;
    asm volatile ("movl %%ebp, %0" : "=r" (ebp));
    return ebp;
}
```
首先再次梳理一下函数调用发生的过程，首先参数压栈，然后call 会把下一条指令的地址压栈，然后进入函数执行，首先保存上一个ebp...
这里的read_eip函数用了__noinline这个关键字，因此这个函数一定会像一个正常的函数调用，read_ebp则是inline，它不会有call,一个函数调用read_ebp时获得的就是该函数调用时ebp寄存器中的值，而eip的获取则是通过调用read_eip，建立一个新的栈帧，asm volatile("movl 4(%%ebp), %0" : "=r" (eip));此时的ebp不再是调用该函数的栈帧的ebp而是新的ebp,新的ebp指向的是调用该函数的栈帧的ebp，新的ebp的高位向上一个地址则是调用该函数的栈帧的eip。
再回到标准答案，这里用的是指针和数组操作，看书332页，指针的加减法取决于该指针所指类型的大小，如
```
int a[10];
int *pa = &a[0];
pa++;
```
由于int占4字节，所以pa++或pa+1以后pa中的地址值加4,且E1[E2]等价于`(*((E1)+(E2)))`，所以`*(pa+2)`等价于pa[2],pa可以像数组名一样使用。