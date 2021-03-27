## Linux之《荒岛余生》（四）I/O篇

我们在cpu篇就提到，iowait高一般代表硬盘到瓶颈了。wait的意思，就是等，就像等正在化妆的女朋友，总是带着一丝焦躁。本篇是《荒岛余生》系列第四篇，I/O篇，计算机中最慢的那一环。其余参见：

[Linux之《荒岛余生》（一）准备篇](https://mp.weixin.qq.com/s?__biz=MzA4MTc4NTUxNQ==&mid=2650519137&idx=1&sn=8922471455cef842b1acbd24db405bca&chksm=8780b1a5b0f738b310c7f74656f1cf51be97f17f9c9b78855afd3c5981ec505584e01ffcf615&token=1441710335&lang=zh_CN&scene=21#wechat_redirect)
[Linux之《荒岛余生》（二）CPU篇](https://mp.weixin.qq.com/s?__biz=MzA4MTc4NTUxNQ==&mid=2650519172&idx=1&sn=e8cade4652e257e8836f52e71d6d9a68&chksm=8780b140b0f73856d66ecb3fd63fe3a5e416d4cb60e5bcbcd907a62b1deb2e9608c76a9666af&token=726684337&lang=zh_CN&scene=21#wechat_redirect)
[Linux之《荒岛余生》（三）内存篇](https://mp.weixin.qq.com/s?__biz=MzA4MTc4NTUxNQ==&mid=2650519204&idx=1&sn=b367c1987fb8c985e83a6cb90f5436a6&chksm=8780b160b0f73876d1998437193d604549b58e310dad44b150e693ac4ff3bb764b110e8e2667&token=18932870&lang=zh_CN&scene=21#wechat_redirect)

# 一点背景

## 速度差异

`I/O`不仅仅是硬盘，还包括外围的所有设备，比如键盘鼠标，比如`1.44M`的`3.5`英寸软盘（还有人记得么）。但服务器环境，泛指硬盘。

硬盘有多慢呢？我们不去探究不同设备的实现细节，直接看它的写入速度（数据有出入，仅作参考）：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cvQbJDZsKLpEyf3ap92UQdxO3OnmQQ3nTFQAicPV5ftRGqbbzKXFeXUH1F7X8OJDm8phoVQv2Qv06MHdvOLoJPg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到普通磁盘的随机写和顺序写相差是非常大的。而`随机写`完全和cpu内存不在一个数量级。缓冲区依然是解决速度差异的唯一工具，所以在极端情况比如断电等，就产生了太多的不确定性。这些缓冲区，都容易丢。

我们举例看一下为了消除这些性能差异，软件方面都做了哪些权衡。

- 数据库设计，采用`BTree`结构组织数据，通过减少对磁盘的访问和随机读取，来提高性能
- `Postgres`通过顺序写`WAL`日志、`ES`通过写`translog`等，通过预写，避免断电后数据丢失问题
- `Kafka`通过`顺序写`来增加性能，但在topic非常多的情况下性能弱化为随机写
- `Kafka`通过`零拷贝`技术，利用DMA绕过内存直接发送数据
- `Redis`使用内存模拟存储，它流行的主要原因就是和硬盘打交道的传统DB速度太慢
- 回忆一下内存篇的`buffer区`，是用来缓冲写入硬盘的数据的。linux的`sync`命令可以将buffer的数据刷到硬盘上，突然断电的话，就不好说了

## 做一个内存盘

如果你的内存够大，那么可以做一个内存盘。跑游戏，做文件交换什么的不要太爽。

```
mkdir /memdisk
mount  -t tmpfs -o size=1024m  tmpfs /memdisk/
```

以上命令划出`1GB`内存，挂载到`/memdisk`目录，然后就可以像使用普通文件夹一样使用它了。只是，速度不可同日而语。

使用dd命令测试写入速度

```
[root@xjj memdisk]# time dd if=/dev/zero of=test.file bs=4k count=200000
200000+0 records in
200000+0 records out
819200000 bytes (819 MB) copied, 0.533173 s, 1.5 GB/s

real    0m0.534s
user    0m0.020s
sys    0m0.510s
```

你见过这么快的硬盘么？

# 排查I/O问题的一般思路

判断`I/O`问题的命令其实并不多，大体有下面几个。

```
#查看wa
top 
#查看wa和io(bi、bo)
vmstat 1
#查看性能相关i/o详情
sar -b 1 2
# 查看问题相关i/o详情
iostat -x 1
# 查看使用i/o最多的进程
iotop
```

## 惊鸿一瞥

首先是我们的老面孔。top、vmstat、sar命令，可以初步判断io情况。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cvQbJDZsKLpEyf3ap92UQdxO3OnmQQ3nQk6qG0Eu14WIvE8JlympW0tb3cDEfbC3CxibxDvBZWul3aUiapMoLFwQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

bi、bo等在你了解磁盘的类型后才有判断价值。我们有更专业的判断工具，所以这些信息一瞥即可。

在本例中，`wa`已经达到`30%`，证明cpu耗费在上面的时间太多。

## 定位问题

如何判断还需要结合`iostat`的帮助。有时候你是无可奈何的，比如这台MySQL的宿主机。你可能会更换更牛X的磁盘，或者整治耗I/O的慢SQL，再或者去改参数。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/cvQbJDZsKLpEyf3ap92UQdxO3OnmQQ3nuTnE0YMibs0BLONkvtKsag0BgYHmicu7kqYynrNmpjCDdqHKX4PhKVlQ/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

你瞧瞧，其实一个`iostat`命令就够了！我们对一些重要结果进行说明：

- **%util** 最重要的判断参数。一般地，如果该参数是100%表示设备已经接近满负荷运行了
- **Device** 表示发生在哪块硬盘。如果你有多快，则会显示多行
- **avgqu-sz** 还记得准备篇里提到的么？这个值是请求队列的饱和度，也就是平均请求队列的长度。毫无疑问，队列长度越短越好
- **await** 响应时间应该低于5ms，如果大于10ms就比较大了。这个时间包括了队列时间和服务时间
- **svctm**  表示平均每次设备`I/O`操作的服务时间。如果`svctm`的值与`await`很接近，表示几乎没有`I/O`等待，磁盘性能很好，如果`await`的值远高于`svctm`的值，则表示`I/O`队列等待太长，系统上运行的应用程序将变慢

总体来说，`%util` 代表了硬盘的繁忙程度，是你进行扩容增加配置的指标。而`await`、`avgqu-sz`、`svctm`等是硬盘的性能指标，如果`%util`正常的情况下反应异常则代表你的磁盘可能存在问题。

> iostat打印出的第1个报告，数值是基于最后一次系统启动的时间统计的；基于这个原因，在大部份情况下，iostat打印出的第1个报告应该被忽略。

另外一种方式就是通过ps命令或者top命令得到状态为D的进程。比如下面命令，循环10次进行状态抓取。

```
for x in `seq 1 1 10`; do ps -eo state,pid,cmd | grep "^D"; echo "----"$x; sleep 5; done
```

## 找到I/O大户

`iostat`查看的是硬盘整体的状况，但我们想知道到底是哪个应用引起的。top系列有一个`iotop`，能够像top一样，看到占用`I/O`最多的应用。`iotop`的本质是一个python脚本，从`proc`目录中获取thread的IO信息，进行汇总。比如

```
[root@xjj ~]# cat  /proc/5178/io
rchar: 628
wchar: 461
syscr: 2
syscw: 8
read_bytes: 0
write_bytes: 0
cancelled_write_bytes: 0
```

如图，显示了当前系统硬盘的读写速度和应用的I/O使用占比。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/cvQbJDZsKLpEyf3ap92UQdxO3OnmQQ3nMNfBicIHbV5OIqaOiaEzBcArf2OesDMFL4vj0HYyyeZL7bwXib6m390rQ/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

那么怎么看应用所关联的所有文件信息呢？可以使用lsof命令，列出了所有的引用句柄。

```
[root@xjj ~]# lsof -p 4050
COMMAND  PID  USER   FD   TYPE             DEVICE     SIZE/OFF      NODE NAME
mysqld  4050 mysql  314u  IPv6          115230644          0t0       TCP iZ2zeeaoqoxksuhnqbgfjjZ:mysql->10.30.134.8:54943 (ESTABLISHED)
mysqld  4050 mysql  320u   REG             253,17         2048  44829072 /data/mysql/mysql/user.MYI
mysqld  4050 mysql  321u   REG             253,17         3108  44829073 /data/mysql/mysql/user.MYD
...
```

更深层的信息，可以通过类似`Percona-Toolkit`这种工具去深入排查，比如`pt-ioprofile`，在此不做详解。

**几个特殊进程说明**

### kswapd0

这依然是由于swap虚拟内存引起的，证明虚拟内存正在大量使用

### jbd2

全称是journaling block driver。这个进程实现的是文件系统的日志功能，磁盘使用日志功能来保证数据的完整性。

可以通过以下方法将其关掉，但一定要权衡

```
dumpe2fs /dev/sda1
tune2fs -o journal_data_writeback /dev/sda1
tune2fs -O "^has_journal" /dev/sda1
e2fsck -f /dev/sda1
```

同时在fstab下重新设定一下，在defaults之后增加

```
defaults,data=writeback,noatime,nodiratime
```

你可能会有一个数量级的性能提升。

# 其他

## 硬盘快满了

使用`df`命令可以看到磁盘的使用情况。一般，使用达到90%就需要重点关注，然后人工介入删除文件了。

```
[root@xjj ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        40G  8.3G   29G  23% /
/dev/vdb1      1008G   22G  935G   3% /data
tmpfs           1.0G  782M  243M  77% /memdisk
```

使用`du`命令可以查看某个文件的大小。

```
[root@xjj ~]# du -h test.file
782M    test.file
```

如果想把一个文件置空，千万不要直接rm。其他应用可能保持着它的引用，经常发上文件删除但空间不释放的问题。比如tomcat的calatina.out文件，如果你想清空里面的内容，不要rm，可以执行下面的命令进行文件内容清空

```
cat /dev/null > calatina.out
```

这非常安全。

## zero copy

kafka比较快的一个原因就是使用了zero copy。所谓的Zero copy，就是在操作数据时, 不需要将数据buffer从一个内存区域拷贝到另一个内存区域。因为少了一次内存的拷贝, CPU的效率就得到提升。

我们来看一下它们之间的区别：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cvQbJDZsKLpEyf3ap92UQdxO3OnmQQ3ndqMTxBCAwMFOnCHTCHL70Guu93eIkLE06gc4vvyy3kVHsx1vatrlHw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



要想将一个文件的内容通过socket发送出去，传统的方式需要经过以下步骤：
=>将文件内容拷贝到内核空间
=>将内核空间的内容拷贝到用户空间内存，比如java应用
=>用户空间将内容写入到内核空间的缓存中
=>socket读取内核缓存中的内容，发送出去

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cvQbJDZsKLpEyf3ap92UQdxO3OnmQQ3nIaaOYwYvwjVc4ibRwssAAOgZ3QwIgJsLVlE1mo0MqibwJwvtTPYDrJeQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上图，zero copy在内核的支持下，少了一个步骤，那就是内核缓存向用户空间的拷贝。即节省了内存，也节省了cpu的调度时间，效率很高。

值得注意的是，java中的zero copy，指的其实是DirectBuffer；而Netty的zero copy是在用户空间中进行的优化。两者并不是一个概念。

## Linux通用I/O模型

面向接口编程？linux从诞生开始就有了。在linux下，一切都是文件，比如设备、脚本、可执行文件、目录等。操作它们，都有公用的接口。所以，编写一个设备驱动，就是实现这些接口而已。

- fd = open(pathname, flags, mode)
- rlen = read(fd, buf, count)
- wlen = write(fd, buf, count)
- status = close(fd)

使用`stat`命令可以看到文件的一些状态。

```
[root@xjj ~]# stat test.file
  File: ‘test.file’
  Size: 819200000     Blocks: 1600000    IO Block: 4096   regular file
Device: 26h/38d    Inode: 3805851     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2018-11-29 12:56:34.801412100 +0800
Modify: 2018-11-29 12:56:35.334415131 +0800
Change: 2018-11-29 12:56:35.334415131 +0800
```

而使用`file`命令，能得到文件的类型信息

```
[root@xjj ~]# file /bin/bash
/bin/bash: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=ab347e897f002d8e3836479e2430d75305fe6a94, stripped
```

## I/O模型

谈完了`I/O`问题的定位方法，就不得不提一下Linux下的`5`种`I/O`模型。等等，这其实是一个网络问题。

- **同步阻塞IO（Blocking IO）** 传统的IO模型
- **同步非阻塞IO（Non-blocking IO）** 非阻塞IO要求socket被设置为NONBLOCK
- **IO多路复用（IO Multiplexing）** 即经典的Reactor设计模式
- **异步IO（Asynchronous IO）** 即经典的Proactor设计模式

java中`nio`使用的就是多路复用功能，也就是使用的Linux的`epoll`库。一般手撸nio的比较少了，大都是直接使用`netty`进行开发。它们用到的，就是经典的reactor模式。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cvQbJDZsKLpEyf3ap92UQdxO3OnmQQ3nTkCGgSkIsIq7ARHtYNF7GBkkic2IfSX0E6YmQyzewJ6IFAbYicI2LwRw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



# 我们能得到什么

除了能够帮助我们评价I/O瓶颈，一个非常重要的点就是：业务研发要合理输出日志，日志文件不仅仅是影响磁盘那么简单，它还会耗占大量的CPU。

对于我们平常的优化思路，也有章可循。像mysql、es、postgresql等，在写真正的数据库文件之前，会有很多层缓冲。如果你对数据可靠性要求并不是那么严重，调整这些缓冲参数的阈值和执行间隔，通常会得到较大的性能提升。

当然，了解I/O还能帮助我们更好的理解一些软件的设计理念。比如leveldb是如何通过LSM来组织数据；ES为什么会存在那么多的段合并；甚至Redis为何存在。

当然，你可能再也无法忍受单机硬盘的这些特性，转而寻求像ceph这样的解决方案。但无论如何，我们都该向所有的数据库研发工作者致敬，在很长一段时间里，我们依然需要和缓慢的I/O共行。