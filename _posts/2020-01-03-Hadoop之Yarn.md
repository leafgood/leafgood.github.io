---
layout: post
title: 'Hadoop之Yarn'
date: 2020-01-03
author: leafgood
color: rgb(255,210,32)
tags: Hadoop入门与进阶
---
# 1.Yarn概述
YARN 是 Hadoop2.x 版本中的一个新特性。
在1.x版本中，MapReduce版本承担了过重的任务，包括资源调度，而到2.x版本中，则将资源调度这部分独立出来了，就是Yarn，这使得hadoop更加稳固，拥有更好的扩展性，可用性，可靠性。

所以我们可以知道，Yarn是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的操作系统平台。


# 2.Yarn 架构
YARN主要由ResourceManager、NodeManager、ApplicationMaster和Container等组件构成。

### 2.1ResourceManager
ResourceManager 拥有系统所有资源分配的决定权，负责协调和管理整个集群（所有NodeManager）的资源，响应用户提交的不同类型应用程序的解析、调度、监控等工作。
ResourceManager 会为每一个 Application 启动一个 MRAppMaster，并且 MRAppMaster分散在各个 NodeManager节点。

ResourceManager（RM）主要作用：
1)处理客户端请求；
2)监控NodeManager
3)启动或监控ApplicationMaster
4)分配与调度资源

### 2.2NodeManager
NodeManager是YARN集群当中资源的提供者，监控应用程序的资源使用情况，通过心跳向集群资源调度器 ResourceManager汇报自己的额状态。
NodeManager监督Container 的生命周期，每个 Container 的资源使用（内存、CPU等）情况，追踪节点健康状况等。
主要作用：
1）管理节点上的资源
2）接收处理ResourceManager的命令
3）执行来自ApplicationMaster的命令


### 2.3Container
Container 容器是一个抽象出来的逻辑资源单位，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等。
Container和集群节点的关系是：一个节点会运行多个Container，但一个Container不会跨节点。
每一个应用程序从ApplicationMaster开始，它本身就是一个container（第0个），一旦启动，ApplicationMaster就会添加任务需求，与Resourcemanager协商更多的container，并且可以动态释放和再申请container。

### 2.4ApplicationMaster

ApplicationMaster负责与scheduler协商合适的container，跟踪应用程序的状态，以及监控它们的进度，ApplicationMaster是协调集群中应用程序执行的进程。每个应用程序都有自己的ApplicationMaster，负责与ResourceManager协商资源（container）和NodeManager协同工作来执行和监控任务 。

# Yarn工作流程
![image](/img/bVcMQbK)

大致流程如下
1）MR程序提交任务,也就是执行这一步
```
main{
  job.waitForCompletion()
 }
```
2）YarnRunner向ResourceManage(RM)r申请一个Application，RM将该应用程序的资源路径返回给YarnRunner。
3）程序拿到资源路径，将运行所需资源提交到hdfs上。
4）程序提交资源完后请运行MrAppMaster。
5）RM将用户的请求封装成一个Task，放到队列中。
6）当NodeManager获取到Task任务，开始创建容器Container，并生成MrAppmaster。
7）Container从HDFS上拷贝资源到本地，MrAppmaster向RM申请运行指定MapTask资源。
8）RM将运行MapTask任务分配到其他NodeManager，NodeManager分别领取任务并创建容器。
9）MR向接收到任务的NodeManager发送程序启动脚本，NodeManager分别启动MapTask。
10) MapTask对数据分区排序,所有MapTask运行完毕后，MrAppMaster向RM申请资源，运行ReduceTask。
11) ReduceTask从MapTask获取相应分区的数据。
12）执行完毕，MR申请注销。















