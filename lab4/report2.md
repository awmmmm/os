#lab4 ex2
> 练习2：为新创建的内核线程分配资源（需要编码）
> 创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：

> 调用alloc_proc，首先获得一块用户信息块。
> 为进程分配一个内核栈。
> 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
> 复制原进程上下文到新进程
> 将新进程添加到进程列表
> 唤醒新进程
> 返回新进程号
> 请在实验报告中简要说明你的设计实现过程。请回答如下问题：

> 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

代码部分有详细注释
先看一下带修改函数的声明部分
```
/* do_fork -     parent process for a new child process
 * @clone_flags: used to guide how to clone the child process
 * @stack:       the parent's user stack pointer. if stack==0, It means to fork a kernel thread.
 * @tf:          the trapframe info, which will be copied to child process's proc->tf
 */
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf)
```
这里的stack其实就是父进程的ESP,如果父进程是内核线程，则stack设0

这里自己没写出来，贴一下答案分析
```
if ((proc = alloc_proc()) == NULL) {
           goto fork_out;
       }

       proc->parent = current;

       if (setup_kstack(proc) != 0) {
           goto bad_fork_cleanup_proc;
       }
       if (copy_mm(clone_flags, proc) != 0) {
           goto bad_fork_cleanup_kstack;
       }
       copy_thread(proc, stack, tf);

       bool intr_flag;
       local_intr_save(intr_flag);
       {
           proc->pid = get_pid();
           hash_proc(proc);
           list_add(&proc_list, &(proc->list_link));
           nr_process ++;
       }
       local_intr_restore(intr_flag);

       wakeup_proc(proc);

       ret = proc->pid;
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
```
这里从从copy_thread开始就蒙圈了，这里个人人认为由于在分配pid、将进程插入链表和散列表时涉及到对共享变量的操作，参考答案用关中断的方法把共享变量保护了起来。而 local_intr_save(intr_flag);中会出现do...while(0)的操作，主要是有关宏展开的问题，[具体参考](https://www.cnblogs.com/Lyush/p/3504559.html)

##思考题（抄的）
> 这个练习提供的注释已经写得非常详细，比实际要写的代码还要多，根据它写出C代码即可，主要实现do_fork。
> 
> 对于pid的分配，内核在do_fork中通过调用get_pid分配一个新的pid然后赋值给新分配的进程。下面分析get_pid的算法。
> 
> 算法主要使用两个变量：next_safe以及last_pid。注意到它们被声明为static，效果类似于全局变量。首先，算法初始化next_safe以及last_pid均为MAX_PID，这是合法pid的上界加一。
> 
> 每次分配，算法首先将last_pid增加1（特别地，若达到或超过MAX_PID则回到1，并认为此时last_pid超过了next_safe），此时若是仍然小于当前next_safe，则立即分配当前last_pid，本次算法结束。否则，此时说明last_pid已到达甚至超过next_safe，需要重新计算next_safe了。
> 
> 为了接下来描述方便，假设所有未被分配的pid组成一段一段区间，叫做空闲pid区间。
> 
> 计算next_safe的主要过程是一个循环，循环内有两个操作“同时”进行：
> 
> 寻找last_pid的下一个合法值，即大于last_pid的最近的空闲区间的下界，存于last_pid。注意，若last_pid被更新，next_safe需要重新开始计算。
> 寻找当前last_pid所在空闲pid区间的上界加一，即已被分配的pid中大于last_pid的最小的那一个，存于next_safe。
> 总之，last_pid维护了最新分配出去的pid，而next_safe维护了last_pid所在空闲pid区间的上界加一。
> 
> 这样就保证了每一次get_pid分配出来的pid没有其他进程正在使用，而且分配效率很高。从这里也看出，需要先分配pid，然后再将进程插入链表，否则算法会混乱。
> 
> 与参考答案对比，由于在分配pid、将进程插入链表和散列表时涉及到对共享变量的操作，参考答案用关中断的方法把共享变量保护了起来，而我没有这么做，应修复。另外，对于多核处理器，应该使用其他手段，因为关中断不能确保其他处理器的互斥访问了。