---
layout: post
title: '03 利用Zookeeper实现Hadoop的HA'
date: 2020-01-05
author: leafgood
color: rgb(255,210,32)
tags: Zookeeper
---

# 1.什么是HA？
所谓HA，也就是高可用，放在实际运营中，那就是7*24小时不间断服务。
那么实现HA，最关键的地方的就是消除单点故障，那什么是单点故障，就没有可替代的节点，一旦这个节点挂了，那整个服务也跟着瘫痪，直到该节点恢复。

# 2.Hadoop HA
Hadoop为什么需要HA呢？
因为存在单点故障，HDFS上的NameNode，这就是一个单点故障，它只存在一个，如果NameNode挂了，或者我们需要对这个节点升级更新，那么会导致集群不可用，
2.0版本之后，还有Yarn也是，ResourceManager也只有一个，也是单点故障。

所以Hadoop需要HA，而且我们要实现两个地方的HA，HDFS的HA和Yarn的HA。

# 3.利用Zookeeper的实现Hadoop的HA
## 3.1 搭建hadoop
参考文档[hadoop部署](https://segmentfault.com/a/1190000038707748) 

## 3.2 搭建Zookeeper集群
参考文档[zookeeper部署](https://segmentfault.com/a/1190000038770862) 

## 3.3 HDFS HA
我这有三个节点，hadoop14、hadoop15、hadoop16
节点信息如下：
```
==============hadoop14 jps================
6096 JobHistoryServer
6436 Jps
5751 DataNode
5929 SecondaryNameNode
6205 NodeManager
5599 NameNode
==============hadoop15 jps================
3616 ResourceManager
3746 NodeManager
4163 Jps
3454 DataNode
==============hadoop16 jps================
3089 Jps
2796 DataNode
2894 NodeManager

```
我们现在实现的是多个namenode和ResourceManager，计划如下：

| server_name | hadoop14 | hadoop15 | hadoop16 |
| ----- | --- |  ----- | ---- |
| namenode |  Y | Y |  |
| ResourceManager|  | Y | Y |

### 3.2.1 HDFS HA
多个NameNode中，只有一个处于Active提供服务，其他的处于Standby状态，作为备用。如果Active的NameNode故障或者该节点需要维护，则切换到Standby节点上提供服务即可，这时提供服务的Standby的节点变为Active。
那么如何做到实时同步节点Active和Standby的元数据信息呢？那就需要一个共享存储系统，我们这里使用Zookeeper。
Active负责向共享存储系统写入元数据，而处于Standby的的节点负责监听，当发现有新数据写入时，则读取这些数据，将其加载到内存中，保证自己内存状态与ActiveNameNode一致。

那么开始配置HDFS HA。
1）core-site.html
```
<!-- 把多个NameNode的地址组装成一个集群mycluster -->

		<property>

			<name>fs.defaultFS</name>

        	<value>hdfs://mycluster</value>

		</property>

	

	<!-- 指定hadoop运行时产生文件的存储目录 -->

		<property>

			<name>hadoop.tmp.dir</name>

			<value>/opt/module/ha/data/tmp</value>

		</property>

   <!-- 声明journalnode服务器存储目录-->
	<property>
		<name>dfs.journalnode.edits.dir</name>
		<value>file://${hadoop.tmp.dir}/jn</value>

	</property>

```

2)hdfs-site.html
```
<!-- 完全分布式集群名称 -->

	<property>

		<name>dfs.nameservices</name>

		<value>mycluster</value>

	</property>

  <!-- NameNode数据存储目录 -->

  <property>

    <name>dfs.namenode.name.dir</name>

    <value>file://${hadoop.tmp.dir}/name</value>

  </property>

 <!-- DataNode数据存储目录 -->

  <property>

    <name>dfs.datanode.data.dir</name>

    <value>file://${hadoop.tmp.dir}/data</value>

  </property>


	<!-- 集群中NameNode节点都有哪些 -->
	<property>
		<name>dfs.ha.namenodes.mycluster</name>
		<value>nn1,nn2,nn3</value>
	</property>


	<!-- nn1的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn1</name>
		<value>hadoop14:9000</value>
	</property>

	<!-- nn2的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn2</name>
		<value>hadoop15:9000</value>
	</property>

	<!-- nn3的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn3/name>
		<value>hadoop16:9000</value>
	</property>

	<!-- nn1的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn1</name>
		<value>hadoop14:9870</value>
	</property>

	<!-- nn2的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn2</name>
		<value>hadoop15:9870</value>
	</property>

	<!-- nn3的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn3/name>
		<value>hadoop16:9870</value>
	</property>


	<!-- 指定NameNode元数据在JournalNode上的存放位置 -->
	<property>
		<name>dfs.namenode.shared.edits.dir</name>
	<value>qjournal://hadoop14:8485;hadoop15:8485;hadoop16:8485/mycluster</value>

	</property>


	<!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
	<property>
		<name>dfs.ha.fencing.methods</name>
		<value>sshfence</value>
	</property>


	<!-- 使用隔离机制时需要ssh无秘钥登录-->
	<property>
		<name>dfs.ha.fencing.ssh.private-key-files</name>
		<value>/home/v2admin/.ssh/id_rsa</value>
	</property>

	<!-- 访问代理类：client用于确定哪个NameNode为Active -->
	<property>	
		<name>dfs.client.failover.proxy.provider.mycluster</name>
	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>
```
3)分别在三个节点启动journalnode服务
```
$HADOOP_HOME/bin/hdfs --daemon start journalnode
```

4)在nn1上，对namenode进行格式化
```
$HADOOP_HOME/binhdfs namenode -format
$HADOOP_HOME/binhdfs --daemon start namenode
```
然后在nn2和nn3上同步nn1的元数据信息。
```
$HADOOP_HOME/binbin/hdfs namenode -bootstrapStandby
```
看下各个节点状态
```
==============hadoop14 jps================
8885 JournalNode
9157 NameNode
7190 QuorumPeerMain
9544 Jps
==============hadoop15 jps================
4579 QuorumPeerMain
5859 NameNode
5940 Jps
5083 JournalNode
==============hadoop16 jps================
3811 JournalNode
4053 NameNode
3305 QuorumPeerMain
4124 Jps
```
有三个namenode启动成功，此刻三个namenode都处于standby状态。
5）启动所有datanode
```
hdfs --daemon start datanode
```
6）将nn1设为Active
```
hdfs haadmin -transitionToActive nn1
```

4)在nn1上，对namenode进行格式化
```
$HADOOP_HOME/binhdfs namenode -format
$HADOOP_HOME/binhdfs --daemon start namenode
```
然后在nn2和nn3上同步nn1的元数据信息。
```
$HADOOP_HOME/binbin/hdfs namenode -bootstrapStandby
```
看下各个节点状态
```
==============hadoop14 jps================
8885 JournalNode
9157 NameNode
7190 QuorumPeerMain
9544 Jps
==============hadoop15 jps================
4579 QuorumPeerMain
5859 NameNode
5940 Jps
5083 JournalNode
==============hadoop16 jps================
3811 JournalNode
4053 NameNode
3305 QuorumPeerMain
4124 Jps
```
有三个namenode启动成功，此刻三个namenode都处于standby状态。
5）启动所有datanode
```
hdfs --daemon start datanode
```
6）将nn1设为Active
```
hdfs haadmin -transitionToActive nn1
```

这个时候还没有结束，因为现在能做的只是手动切换Active和Standby，还没有实现namenode自动切换，
实现自动切换就要借助ZooKeeper和ZKFailoverController（ZKFC）两个新组件，我们先简单说下这两个组件的作用，再进行配置。

ZooKeeper维护少量协调数据，通知客户端这些数据的改变和监视客户端故障的高可用服务，Zookeeper有以下功能：
a)故障检测：ZooKeeper中为集群中的每个NameNode维护了一个持久会话，如果节点挂了，那ZooKeeper中的会话将终止，ZooKeeper通知另一个NameNode需要触发故障转移。
b)Active NameNode选择：ZooKeeper提供了一个简单的ActiveNameNode选择机制，用于唯一的选择一个节点为active状态，其实就是锁机制。

ZKFC是ZooKeeper的客户端，监视和管理NameNode的状态，它是实现HA的另一个组件。每个运行NameNode的主机上也运行了一个ZKFC进程，ZKFC负责：
a）健康监测：我们需要知道NameNode的健康状态，ZKFC定期地ping与之在相同主机的NameNode，只要该NameNode及时地回复健康状态，ZKFC认为该节点是健康的，如果该节点挂了，冻结或进入不健康状态，健康监测器标识该节点为非健康的。
b）ZooKeeper会话管理：当本地NameNode是健康的，ZKFC保持一个在ZooKeeper中打开的会话。如果本地NameNode处于active状态，ZKFC也保持一个特殊的znode锁，该锁使用了ZooKeeper对临时节点的支持，如果会话终止，锁节点将自动删除。
c）基于ZooKeeper的选择：如果本地NameNode是健康的，且ZKFC发现没有其它的节点当前持有znode锁，它将为自己获取该锁。如果成功，则它已经赢得了选择，并负责运行故障转移进程以使它的本地NameNode为Active。故障转移进程与前面描述的手动故障转移相似，首先如果必要保护之前的现役NameNode，然后本地NameNode转换为Active状态。

好了，了解完这两个组件，继续进行我们的配置


7）hdfs-site.xml中增加
```
<property>
	<name>dfs.ha.automatic-failover.enabled</name>
	<value>true</value>
</property>
```
8）core-site.xml增加
```
<property>
	<name>ha.zookeeper.quorum</name>
	<value>hadoop14:2181,hadoop15:2181,hadoop16:2181</value>
</property>
```
9）启动
9-1）关闭所有hdfs服务
9-2）在每个节点上启动Zookeeper服务，生成环境中可自己写群启服务的脚本。
9-3）在每个节点上初始化Zookeeper中HA的状态 
```
hdfs zkfc -formatZK
```
9-4）启动HDFS服务
```
start-dfs.sh
```
至此配置完成了。
我们在web登录，可以看到一个Active，两个Standby。
我们也可以进行验证是否成功，kill掉Active节点，看看切换是否成功。

可能会要的问题
```
No Route to Host from  hadoop......
```
这个一般是防火墙的关系，关掉防火墙一般可以了，这个不过多赘述。




### 3.2.2 Yarn HA
1.yarn-site.xml
```
<property>

        <name>yarn.nodemanager.aux-services</name>

        <value>mapreduce_shuffle</value>

    </property>



    <!--启用resourcemanager ha-->

    <property>

        <name>yarn.resourcemanager.ha.enabled</name>

        <value>true</value>

    </property>

 

    <!--声明两台resourcemanager的地址-->

    <property>

        <name>yarn.resourcemanager.cluster-id</name>

        <value>cluster-yarn1</value>

    </property>



    <property>

        <name>yarn.resourcemanager.ha.rm-ids</name>

        <value>rm1,rm2</value>

    </property>



    <property>

        <name>yarn.resourcemanager.hostname.rm1</name>

        <value>hadoop15</value>

    </property>



    <property>

        <name>yarn.resourcemanager.hostname.rm2</name>

        <value>hadoop16</value>

    </property>

 

    <!--指定zookeeper集群的地址--> 

    <property>

        <name>yarn.resourcemanager.zk-address</name>

        <value>hadoop14:2181,hadoop15:2181,hadoop16:2181</value>

    </property>



    <!--启用自动恢复--> 

    <property>

        <name>yarn.resourcemanager.recovery.enabled</name>

        <value>true</value>

    </property>

 

    <!--指定resourcemanager的状态信息存储在zookeeper集群--> 

    <property>

        <name>yarn.resourcemanager.store.class</name>     <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>

</property>

```
2.启动yarn
```
start-yarn.sh
```
3)查看服务状态
```
[v2admin@hadoop14 ha]$ yarn rmadmin -getServiceState rm1
standby
[v2admin@hadoop14 ha]$ yarn rmadmin -getServiceState rm2
active
```
至此Hadoop的HA配置完成了。