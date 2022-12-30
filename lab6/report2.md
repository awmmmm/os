#lab6 ex2
首先需要换掉RR调度器的实现，即用default_sched_stride_c覆盖default_sched.c。然后根据此文件和后续文档对Stride度器的相关描述，完成Stride调度算法的实现。

后面的实验文档部分给出了Stride调度算法的大体描述。这里给出Stride调度算法的一些相关的资料（目前网上中文的资料比较欠缺）。
具体从实验指导书3.6.1开始

##思考题的copy
太笨想不出来，直接抄
[答案](https://github.com/cjspark/BlackFriday-Analysis)

这里自己写的遇到的一些小问题，比如初始化PCB,time_slice不能设置为1,函数的调用参数与返回值的一些小问题，简单的边界bug等等。