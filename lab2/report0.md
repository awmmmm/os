# LAB2 EX0

##练习0：填写已有实验

本实验依赖实验1。请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。提示：可采用diff和patch工具进行半自动的合并（merge），也可用一些图形化的比较/merge工具来手动合并，比如meld，eclipse中的diff/merge工具，understand中的diff/merge工具等。

##linux下的diff和patch命令使用

首先用diff找出差异保存到patch中，然后用

单文件
```
$ diff -u ori.c tar.c > update.patch

```
多文件
```
$ diff -uprN linux/ori/ linux/tar/ > all_update.patch

$ diff -Naur test1.txt test2.txt > test.patch
```
diff参数解释
-N 在比较目录时如果某个文件只出现了一次，那么在比较不同时会默认和空文件比较
-a 将所有的文件都作为普通text(之比较文本文件)
-u 以合并的方式显示文件内容的不同
-r 如果是文件夹则进行递归进行比较
```
$ patch -p0 < test.patch
$ patch ori.c < update.patch
```
使用patch指令将文件"testfile1"升级，其升级补丁文件为"testfile.patch"，输入如下命令：
```
$ patch -p0 testfile1 testfile.patch    #使用补丁程序升级文件
```


参数介绍
patch命令中最常用的就是-pX这个参数

####假设patch文件如下内容：

--- test1.txt   2018-08-01 13:17:33.530350672 +0800
+++ test2.txt   2018-08-01 13:18:54.326350260 +0800
@@ -1 +1,2 @@ 
 aaaa
+bbbb

####注意到patch文件如下内容

--- test1.txt   2018-08-01 13:17:33.530350672 +0800

此时我们的参数为-p0，此时patch 就会在当前目录下寻找test1.txt文件，如在在patch文件中是这样记录的

---a/b/test1.txt   2018-08-01 13:17:33.530350672 +0800

那么-p0会在当前目录下寻找a目录，a目录下寻找b,之后在b中寻找test1.txt文件。
如果是 -p1，patch命令就会舍弃a，先寻找b再寻找test1.txt
如果是-p2 ,会舍弃a/b，直接寻找test1.txt
所以-pX中 X代表就是所要舍弃的层级目录
patch还有很多参数，但是-pX是最为常用的


作者：狗钱偷生
链接：https://www.jianshu.com/p/1df286850317
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。