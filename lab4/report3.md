#lab4 ex3
> 练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。（无编码工作）
> 请在实验报告中简要说明你对proc_run函数的分析。并回答如下问题：
> 在本实验的执行过程中，创建且运行了几个内核线程？
> 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?请说明理由
> 完成代码编写后，编译并运行代码：make qemu
> 如果可以得到如 附录A所示的显示内容（仅供参考，不是标准答案输出），则基本正确。

总体的线程执行流程是首先在proc_init中初始化第“0”个线程，并将current指向这第0个线程。然后初始化（准备好）第一个线程，通过fork的方式，即复制第0个，再对其做修改。

然后在一系列的初始化工作准备好之后，调用cpu_idle().开始循环调用schedule，切换到第1个线程，主要就是调用的proc_run这个函数，直接放代码
```
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}

```
首先禁用中断，以免线程切换期间产生中断，发生错误，然后current切换到新的，加载新线程的内核栈到TSS,（方便后续特权级切换时找到内核栈），更新新页表，调用switch_to，此时就已就切换完成并开始执行了，这段汇编完成了线程0上下文（及将当前的寄存器保存到PCB的content里）保存，新线程的加载，(将新PCB的content回复到各个寄存器)此时的esp就发生了变换（栈的切换），并在汇编的最后往栈压入了ret后要执行的指令。后面的流程在之前也说过了。这里有个小坑（piazza上看到的）


> ①在进行switch_to操作时，ret时，已经将运行环境的8个寄存器从from进程的换成了to进程的，因此在ret后直接跳转到forkret，进而iret到kernel_thread_entry处开始运行main，那么上述代码中的最后一行将迟迟不能执行，也就是说中断被关闭后迟迟不能开启，导致中断堵塞，直到main进程执行do_exit操作。这是合理的吗？或者是我理解错了？

> A:对于第一个问题，我的理解是，forkret在iret处会有中断现场的恢复。然后就开始正常执行了。请实际跟踪系统在这段的行为，然后回复结果或继续置疑。
> 第一个问题，在创建新线程do_fork中，会将tf中的eflags设置成允许中断。所以在forkret时，会恢复eflags寄存器，就能正常执行了

还有个小注意：这里每个栈空间分的不是很多，一旦溢出就容易被有心之人进行利用