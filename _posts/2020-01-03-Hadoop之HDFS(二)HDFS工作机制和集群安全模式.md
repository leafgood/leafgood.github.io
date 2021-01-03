---
layout: post
title: 'Hadoop之HDFS(二)HDFS工作机制和集群安全模式'
date: 2020-01-03
author: leafgood
color: rgb(255,210,32)
tags: Hadoop入门与进阶
---
## 1 NameNode和DataNode之间的心跳机制
1）NameNode启动时，会启动一个IPC server服务，
2）DataNode启动后会主动连接NameNode的IP server服务，默认每隔3秒连接一次，也就是心跳。
这个时间可以通过 dfs.heartbeat.interval参数设置，也就是心跳时间。
3）DataNode通过心跳在NameNode注册，NameNode通过心跳获取DataNode的状态和NameNode下达的操作指令，同时周期性的向NameNode汇报自己所有的块信息。
4）当NameNode长时间没有收到DataNode的心跳，就认为DataNode挂掉了。
这种心跳机制同样存在于Yarn中ResourceManager和NodeManager中。

这个就是Hadoop的Master/Slave架构，NameNode和ResourceManager就是Master，DataNode和NodeManager就是Slave。

## 2 NameNode和SecondaryNameNode的工作机制
第一个问题，NameNode元数据怎样保存的？
首先NaomNode的元数据需要放在内存中，因为我们需要经常访问NameNode节点获取元数据，若是放在磁盘中，那效率会非常低。

既然放在内存中，那必要要有一个机制保证内存数据的安全，因为内存中数据一旦断电就丢了，所以内存中的元数据也必须要落地到磁盘，这个就是FstImage.

但这样还没有到万事大吉的地步，内存中的元数据随时可能更新，这时是否要同步更新FsImage呢？如果我们更新，必然会导致效率底下，如果我们不更新，那内存中的元数据和FsImage就会不一致，一旦出现NameNode节点断电之类的情况，就会出现部分数据丢失。

那么我们引入这样一个记录文件Edits，只要内存中的元数据增加或者更新，那么就同步把这个操作记录追加到Edits，这样即便NameNode断电，我们还可以根据Edits和FsImage来恢复元数据。

新的问题又来了，内存中的元数据可是经常发生变化的，那不断的追加记录到Edits中，那必然会导致这个文件越来越大，那么未来我们需要恢复元数据时，需要花费的时间也必然大大增加，影响我们效率，所以我们需要定期对FsImage和Edits进行合并。

好了，任务来了，定义把FsImage和Edits进行合并，那这个任务谁来做呢？NameNode可以吗？当然可以，但这会导致NameNode任务过重，影响效率，那为了保证效率，就把这个任务交给另外一个人来做，那就是Secondary NameNode。

从这里可以明白Secondary NameNode并不是NameNode的热备，当NameNode挂了的时候，它并不能替代NameNode工作，但它可以用帮助恢复NameNode。

具体NameNode和Secondary NameNode的工作流程如下
阶段1：
1）首次启动集群后，我们需要对NameNode格式化，这时会创建FsImage和Edits，这些文件就在$HADOOP_HOME/data/name/current下.
之后启动，直接加载Edits和FsImage到内存中。
2）Client也就是客户端对元数据进行增删改的操作请求。
3）NameNode先记录操作，更新日志，然后在内存中对元数据进行增删改的操作。

阶段2：
SecondaryNameNode执行合并的操作，叫CheckPoint，这个操作有两个触发条件。
第一个，就是间隔时间到，默认是1小时，这个可以调整。
第二个，就是SecondaryNameNode会一分钟检查一次操作次数，当操作数达到设置的上限，就会触发。

1）首先SecondaryNameNode会询问NameNode是否需要执行CheckPoint
2）拿到NameNode的返回结果，就开始请求执行CheckPoint
3）NameNode滚动更新正在的Edits日志，将滚动前的Edits和FsImage文件拷贝到SecondaryNameNode上。
4）SecondaryNameNode将两个文件加载到内存进行合并，生成新的fsImage.chkpoint，拷贝到NameNode上。
5）NameNode将fsimage.chkpoint命名为fsimage。

当然在有了HA后，也很少使用SecondaryName了。


## 3.DataNode如何保证存储数据的完整性
我们知道数据是存储在DataNode节点中，那如果某个DataNode节点的上数据损坏，譬如压缩包，DataNode怎样处理这个问题，来保障数据的完整性？

一般我们为了保障数据的完整性，都是采用数据校验技术，常见的有：
1）奇偶校验
2）md5、sha1等校验
3）CRC_32循环冗余校验

HDFS能够通过io.bytes.per.checksum属性设置校验方式。

在写入数据时，客户端将数据和校验一起发送给DataNode，位于最后的DataNode节点负责校验数据，如果数据存在错误，那么客户端会接收到一个ChecksumException异常。

在读取数据时，客户端会进行校验，与DataNode中存储的校验和进行比较，如果存在错误，那么它会报告个NameNode，再抛出ChecksumException异常。Namenode将这个数据块复本标记为损坏，之后，它不会再将处理请求发送到这个节点。
随后，它安排这个数据块的一个复本复制到这个DataNode，损坏的数据块删除。

另外，DataNode节点会在后台执行一个线--DataBlockscanner（数据块检測程序）周期性的验证存储在数据节点上的全部块。

这个就是HDFS保证数据完整性的机制。 

## 4.DataNode掉线时限的参数设置
有哪些情况会引发DataNode掉线？
譬如网络故障、DataNode进程挂了或者服务器掉电等等情况。

当发生这些事件时。NameNode不会立即判定这个DataNode挂了，而是要等待一段时间，这个时长就是超时时长。
HDFS默认的超时时长为10分钟30秒，那么这个个时间怎么来的呢？
它有一个计算公式：
timeout = 2 * heartbeat.recheck.interval + 10 *dfs.heartbeat.interval

timeout表示超时时长，heartbeat.recheck.interval默认为5分钟，dfs.heartbeat.interval默认为3秒，计算结果就是10分30秒。

## 5 安全模式
前面讲了，当NameNode启动后，首先是将FsImage和Edits文件加入内存，这个是为保证得到最新的元数据，这个其实也是合并，之后生成一个新的FsImage和一个空白的Edits，然后启动IPC Server服务，监听DataNode的请求，在这个期间，NameNode的文件系统对外界处于只读状态，也就是安全模式。

之后DataNode启动，在各个DataNode通过NameNode的IPCServer 发送他们最新的块列表信息。

当达到dfs.replication.min 设定的值，NameNode会退出安全模式，这个参数设置的值就是最小副本条件，指的是文件系统中块满足的最小副本级别。


