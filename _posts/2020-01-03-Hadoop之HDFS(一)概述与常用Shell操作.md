---
layout: post
title: 'Hadoop之HDFS(一)概述与常用Shell操作'
date: 2020-01-03
author: leafgood
color: rgb(255,210,32)
tags: Hadoop入门与进阶
---
# 1.HDFS概述
## 1.1 HDFS简介
HDFS(Hadoop Distributed File System) ，Hadoop分布式文件系统，用来解决海量数据的存储问题。

## 1.2 HDFS的特点
#### 优势
- 高容错性：HDFS多副本分布式存储，当一个副本丢失了，能够自动恢复，所以HDFS具有高容错性，默认是3副本。

- 大数据处理：HDFS处理数据的规模甚至可以达到PB级别，文件数量甚至百万之上
- HDFS是设计成适应一次写入，多次读出的场景，且不支持文件的修改

#### 劣势
- HDFS不擅长大量小文件的存储，因为NameNode的内存是有限的，但大量小文件存储，会耗用大量NameNode的内存，来存储文件的目录、块信息等。
- HDFS高延时，不适合低延时的访问
- HDFS不支持并发写和文件的修改

## 1.3 HDFS架构图解
![hdsf1](../assets/article/hdfs1) 

1）NameNode（nn）：它是一个主管、管理者，Master节点，负责维护整个hdfs文件系统的目录树，以及每一个路径（文件）所对应的block块信息。
（1）管理HDFS的名称空间；
（2）配置副本策略；
（3）管理数据块（Block）映射信息；
（4）处理客户端读写请求。

2）DataNode：就是Slave。NameNode下达命令，DataNode执行实际的操作，负责存储client发来的数据块block；执行数据块的读写操作。Datanode是HDFS集群从节点，每一个block都可以在多个datanode上存储多个副本，副本数量也可以通过参数dfs.replication设置。
HDFS存储文件是**分块存储（block）**，默认大小在hadoop2.x版本中是128M，老版本中是64M。这个可以通过配置参数dfs.blocksize来进行调整。HDFS块的大小设置主要取决于磁盘传输速率，

3）Secondary NameNode：充当小弟的角色。
（1）辅助NameNode，分担其工作量，比如定期合并Fsimage和Edits，并推送给NameNode ；
（2）在NameNode挂掉时，可辅助恢复NameNode，不过现在有HA，所以很少使用Secondary NameNode。

4）Client：顾名思义，就是客户端。
（1）文件切分。Client将文件切分成一个一个的Block，然后进行上传到HDFS上。
（2）与NameNode交互，获得读取或写入文件的位置信息；
（3）与DataNode交互，读写数据；
（4）Client提供一些命令管理HDFS，比如NameNode格式化操作；
（5）Client提供一些命令访问HDFS，比如对HDFS增删查改操作；

# 2.HDFS Shell命令
## 2.1 语法操作
1）hadoop fs 命令
2）hdfs dfs 命令

## 2.2 常用命令
hadoop的shell命令使用起来非常简单，和linux Shell命令十分相似。

- 1.启动集群
```
sbin/start-dfs.sh
sbin/start-yarn.sh
```
- 2.查看命令帮助
```
hadoop fs -help ls
```

- 3.查看系统系统目录
```
[v2admin@hadoop10 sbin]$ hadoop fs -ls /
2020-12-29 10:20:58,645 INFO  [main] Configuration.deprecation (Configuration.java:logDeprecation(1395)) - No unit for dfs.client.datanode-restart.timeout(30) assuming SECONDS
Found 7 items
drwxr-xr-x   - v2admin supergroup          0 2020-12-23 20:37 /flume
drwxr-xr-x   - v2admin supergroup          0 2020-12-28 17:23 /hbase
drwxr-xr-x   - v2admin supergroup          0 2020-12-23 14:43 /home

```
- 4.创建文件夹
```
[v2admin@hadoop10 sbin]$ hadoop fs -mkdir /student
```
- 5.多级文件夹创建
```
[v2admin@hadoop10 sbin]$ hadoop fs -mkdir /student/zhangsan/course_count
```

- 6.上传和下载文件
从本地拷贝文件至hdfs
```
[v2admin@hadoop10 sbin]$ cd /home/v2admin/demo
[v2admin@hadoop10 demo]$touch words.txt
[v2admin@hadoop10 demo]$echo aaa > words.txt
[v2admin@hadoop10 demo]$ hadoop fs -put words.txt /student
// words 是我本地的文件  后面跟上传hdfs的目录
[// hadoop fs -copyFromLocal words.txt /student
// 等同于put
[v2admin@hadoop10 demo]$ hadoop fs -ls /student
2020-12-29 10:32:34,660 INFO  [main] Configuration.deprecation (Configuration.java:logDeprecation(1395)) - No unit for dfs.client.datanode-restart.timeout(30) assuming SECONDS
Found 2 items
-rw-r--r--   3 v2admin supergroup          4 2020-12-29 10:31 /student/words.txt

```
从本地剪贴到HDFS
```
[v2admin@hadoop10 demo]$ touch bbb.txt
[v2admin@hadoop10 demo]$ ls
bbb.txt  f  pos.log  words.txt
[v2admin@hadoop10 demo]$ hadoop fs -moveFromLocal bbb.txt /student
2020-12-29 10:40:20,634 INFO  [main] Configuration.deprecation (Configuration.java:logDeprecation(1395)) - No unit for dfs.client.datanode-restart.timeout(30) assuming SECONDS
[v2admin@hadoop10 demo]$ ls
f  pos.log  words.txt
[v2admin@hadoop10 demo]$ hadoop fs -ls /student
2020-12-29 10:40:40,097 INFO  [main] Configuration.deprecation (Configuration.java:logDeprecation(1395)) - No unit for dfs.client.datanode-restart.timeout(30) assuming SECONDS
Found 3 items
-rw-r--r--   3 v2admin supergroup          0 2020-12-29 10:40 /student/bbb.txt
-rw-r--r--   3 v2admin supergroup        151 2020-12-21 14:45 /student/stu.txt
-rw-r--r--   3 v2admin supergroup          4 2020-12-29 10:31 /student/words.txt
```
追加文件到已经存在的文件末尾
```
[v2admin@hadoop10 demo]$ echo bbb > bbb.txt
[v2admin@hadoop10 demo]$ hadoop fs -appendToFile bbb.txt /student/words.txt
```
那么下载words.txt文件，看看文件发生什么变化吧
```
[v2admin@hadoop10 demo]$ mkdir download
[v2admin@hadoop10 demo]$ hadoop fs -get /student/words.txt ./download/
[v2admin@hadoop10 demo]$ cat download/words.txt 
aaa
bbb
```
下载文件除了get还有其他两个
```
[v2admin@hadoop10 demo]$ hadoop fs -copyToLocal /student/words.txt ./download/
// 等同于get

[v2admin@hadoop10 demo]$ hadoop fs -getmerge /student/* ./download/
// 合并下载多个文件
```
- 7.从HDFS一个路径拷贝到HDFS另一个路径
```
[v2admin@hadoop10 demo]$hadoop fs -mkidr /newStu
[v2admin@hadoop10 demo]$hadoop fs -cp /student/bbb.txt /newStu
```
- 8.在HDFS上移动文件
```
[v2admin@hadoop10 demo]$hadoop fs -mv /student/words.txt /newStu
```

- 9.在HDFS上删除文件
```
// 常规删除
[v2admin@hadoop10 demo]$hadoop fs -rm /newStu/bbb.txt

// 强制删除
[v2admin@hadoop10 demo]$hadoop fs -rm -r /newStu
```
- 10 查看文件内容
```
[v2admin@hadoop10 demo]$hadoop fs -cat /student/bbb.txt
```

更多命令参考[http://hadoop.apache.org/docs/r3.1.3/hadoop-project-dist/hadoop-common/FileSystemShell.html](http://hadoop.apache.org/docs/r3.1.3/hadoop-project-dist/hadoop-common/FileSystemShell.html) 











