#lab4 ex1
###PS：一丢丢的感想
之前一直困惑的debug问题总算解决了，哈哈多少有点无语了，直接调make debug就可以了，gdb确实是一个非常强大的调试工具，随时监控变量&寄存器的变化。这里因为自己的愚蠢浪费了好多时间在无用的问题上。但可以把一些有用的东西整理一下下

###iret指令&中断用于任务切换和TSS的一丢丢简单了解

首先ret和iret一样是要出栈栈顶的东西到相关寄存器的，从而实现指令的跳转等一系列功能。首先明确一点，不管ret还是iret,一定会出栈栈顶作为新的EIP,而本次实验中用到的和之前中断返回时用到的是根据IRET出栈的CS判断CS的特权和当前代码段CS(?不确定)之间的权限是否有变化，有的话还要弹出并刷新ESP和SS寄存器。


TSS的一点了解（抄的）
[大佬的解释](https://blog.csdn.net/x13262608581/article/details/125941627?spm=1001.2101.3001.6650.9&utm_medium=distribute.wap_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-9-125941627-blog-7836854.wap_blog_relevant_default&depth_1-utm_source=distribute.wap_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-9-125941627-blog-7836854.wap_blog_relevant_default)

我的粗钱理解，个人认为对于本实验，只要知道中断产生特权级切换后就可以通过TSS找到用户进程对应的内核栈（SS0和ESP0）
###实验部分
哈哈，其实实验真的灰常简单，但我被一些愚蠢的操作问题和由于对一些概念的不了解折磨了整整3天，还是进个群，遇到问题有人帮忙解决，不管工作还是学习效率高一些

练习1：分配并初始化一个进程控制块（需要编码）

alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。

    【提示】在alloc_proc函数的实现中，需要初始化的proc_struct结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name。

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

    请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）
灰常简单，memset把分配给TCB\PCB的内存置0，然后把标志设到你想设的初始值就可以了

至于问题content和trapframe的作用，content很好理解,就是为了保存进程执行时的各个寄存器信息。具体看switch.S切换就知道了
至于trapframe俺和这位老哥一样，起初觉得有多此一举的行为
> Lab4 疑问
依我的理解，在lab4进程切换的时候，有以下几个步骤：

    1）利用proc_struct中的context的eip，跳到forkret

    2）在forkret中，利用原先设计好的context.esp，将栈顶指向proc_struct中的trapframe

    3）跳到trapret，将trapframe中的寄存器一一还原，将eip指向kernel_thread_entry

> 我的疑问就是，为什么要先用context还原一遍寄存器，然后到trapframe里再还原一遍？

> 为什么不能直接将context中的eip就指向kernel_thread_entry？

> 一直搞不懂proc_struct中trap的作用，求解释！
![](../图片/图片8.png)

##PS:目前的一些疑惑
```
.globl __trapret
__trapret:
    # restore registers from stack
    popal

    # restore %ds, %es, %fs and %gs
    popl %gs
    popl %fs
    popl %es
    popl %ds

    # get rid of the trap number and error code
    addl $0x8, %esp
    iret

.globl forkrets
forkrets:
    # set stack to this new process's trapframe
    movl 4(%esp), %esp
    jmp
```
这里让esp先指到trapframe，然后出栈刷新寄存器，iret后执行预先在中断帧tf->eip保留好的代码。以下是我的疑惑：

Q1:iret这里可以理解是无特权级切换的中断返回，即iret以后esp指向tf->tf_esp?请问这里的理解正确吗？

Q2:如果我的Q1理解正确，那么接下来就是执行线程的函数，此时的esp指向的tf_esp，岂不是占用了预先保留的中断帧？万一后续有中断产生是不是会有影响？

A1:yes
A2:不知道