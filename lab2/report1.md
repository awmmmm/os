# LAB2 EX1
###练习1：实现 first-fit 连续物理内存分配算法（需要编程）

在实现first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改default_pmm.c中的default_init，default_init_memmap，default_alloc_pages， default_free_pages等相关函数。请仔细查看和理解default_pmm.c中的注释。

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

你的first fit算法是否有进一步的改进空间

#####基础实验，但是自己太菜，踩了不少坑。
简单看下first_fit的原理，原理很简单
![](../图片/firsrfit.png)
然后简单介绍一下完成实验用到的一些结构体类型和一些重要参数

首先，bootloader的工作有增加，在bootloader中，完成了对物理内存资源的探测工作（可进一步参阅附录A和附录B），让ucore kernel在后续执行中能够基于bootloader探测出的物理内存情况进行物理内存管理初始化工作。

具体的物理内存探测参考附录
然后首先实现简单的连续物理内存块的分配算法，首先要**以页为单位**管理物理内存
```
struct Page {
    int ref;        // page frame's reference counter
    uint32_t flags; // array of flags that describe the status of the page frame
    unsigned int property;// the num of free block, used in first fit pm manager
    list_entry_t page_link;// free list link
};
```
具体还是看附录**以页为单位管理物理内存**

然后这里用到了一个很方便的双向循环链表的数据结构
```
struct list_entry {
    struct list_entry *prev, *next;
};
```
使用方法：
```
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```
这里的freelist是双向链表的头指针，存的是空闲内存快，即上面page结构的pagelink变量，然后可以通过
```
struct Page *p = le2page(le, page_link);
```
le2page把一个链表元素入口类型的pagelink转换为page结构体，这么做的好处是通过指针访问，那么链表中存的东西不一定需要类型相同，只需要在类型的成员变量中声明一个list_entry结构就可以和别的数据类型一起存放。

然后是page_mananger
```
struct pmm_manager {
            const char *name; //物理内存页管理器的名字
            void (*init)(void); //初始化内存管理器
            void (*init_memmap)(struct Page *base, size_t n); //初始化管理空闲内存页的数据结构
            struct Page *(*alloc_pages)(size_t n); //分配n个物理内存页
            void (*free_pages)(struct Page *base, size_t n); //释放n个物理内存页
            size_t (*nr_free_pages)(void); //返回当前剩余的空闲页数
            void (*check)(void); //用于检测分配/释放实现是否正确的辅助函数
};
```
主要使用函数指针的方式来多手段方便的管理内存。

###一些坑
这里debug不了，然后因为freepage逻辑没搞对，有些情况没考虑周全，浪费了很多时间，代码本身没啥难度，至于改进暂时想不到，后面有bug的话再和答案比对一下。