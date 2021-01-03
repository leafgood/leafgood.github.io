---
layout: post
title: '05Hadoop之MapReduce(一)初识'
date: 2020-01-03
author: leafgood
color: rgb(255,210,32)
tags: Hadoop入门与进阶
---

## 1.MapReduce介绍
MapReduce是一个分布式运算程序的编程框架，是用户开发“基于Hadoop的数据分析应用”的核心框架。
MapReduce核心功能是将用户编写的业务逻辑代码和自带默认组件整合成一个完整的分布式运算程序，并发运行在一个Hadoop集群上。
MapReduce的作用就是大数据处理。

## 2.MapReduce的核心思想
举一个例子，我们有1000个球，里面有红色、蓝色、白色各种颜色的球，我们想知道里面有多少红球，我们把这个找出红球数量的任务交给了MapReduce来做。

1）MapReduce会将这1000个球交分成若干份，交给若干人
2）每人数自己手中红球的数量，然后上报上去
3）汇总所有数，求和，得到红球数量。

在大数据处理上，MapReduce设计思想的核心就是"分而治之"，它将复杂的、运行在大规模集群上的并行计算过程高度抽象成两个Map和Reduce两个函数。





| 函数名 | 输入 | 处理 | 输出 |
| --- | --- | --- | --- |
| Map  |  键值对\<k1,v1\> | Map 函数将处理这些键值对，并以另一种键值对形式输出中间结果 List(\<K2,V2\>) | List(\<K2,V2\> |
| Reduce | List(\<K2,V2\> | 对传入的中间结果列表数据进行某种整理或进一步的处理，并产生最终的输出结果List(\<K3,V3\> | List(\<K3,V3\> |

简单来说，Map阶段负责拆分处理，Reduce阶段将Map阶段的处理结果进行汇总

## 3.官放wordcount示例
这里有一个words.txt文件
```
[v2admin@hadoop10 demo]$ cat words.txt 
zhangsan hello tianjin
beijing shanghai shandong
hebei guangzhou tangshan
```
上传至hadoop上
```
[v2admin@hadoop10 demo]$ hadoop fs -put words.txt /home/input
```
运行wordcount
```
// 进入$HADOOP_HOME/share/hadoop/mapreduce
[v2admin@hadoop10 mapreduce]$ hadoop jar hadoop-mapreduce-examples-3.1.3.jar wordcount /home/input /home/output1
```
然后我们看下过程
```
2020-12-30 14:31:30,989 INFO  [main] mapreduce.Job (Job.java:monitorAndPrintJob(1647)) -  map 0% reduce 0%
2020-12-30 14:31:36,035 INFO  [main] mapreduce.Job (Job.java:monitorAndPrintJob(1647)) -  map 100% reduce 0%
2020-12-30 14:31:41,134 INFO  [main] mapreduce.Job (Job.java:monitorAndPrintJob(1647)) -  map 100% reduce 100%
```
跟上述所说一样，任务被分成两个阶段map和reduce，我们看下最终结果
```
[v2admin@hadoop10 mapreduce]$ hadoop fs -cat /home/output1/part-r-00000
2020-12-30 14:34:52,772 INFO  [main] Configuration.deprecation (Configuration.java:logDeprecation(1395)) - No unit for dfs.client.datanode-restart.timeout(30) assuming SECONDS
2020-12-30 14:34:53,299 INFO  [main] sasl.SaslDataTransferClient (SaslDataTransferClient.java:checkTrustAndSend(239)) - SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
beijing	1
guangzhou	1
hebei	1
hello	1
shandong	1
shanghai	1
tangshan	1
tianjin	1
zhangsan	1
```
## 4 自己去实现一个wordcount
先看下官方的wordcount，通过反编译工具可以去看源码，要实现一个wordcount，需要编写三部分内容：Mapper、Reducer和Driver。
同时也看到hadoop中并没有使用java中的数据类型，而是有自己实现的。

| Java类型	|  Hadoop Writable类型 |
| --- | --- |
Boolean |	BooleanWritable
Byte |	ByteWritable
Int	| 	IntWritable
Float	|	 FloatWritable
Long	|	LongWritable
Double	|	DoubleWritable
String	|	Text
Map	|	MapWritable
Array	|	ArrayWritable

- Mapper的示例代码：
```java
/**
 * Map编写步骤
 * 1）自定义的Mapper类要继承Mapper类
 * 2）Mapper的输入数据为k-v对形式，类型自定义
 * 3）实现父类map()方法，也是业务逻辑
 * 4）输出结果也是kv-对,类型自定义
 *
 */
public class WordCountMap extends Mapper<LongWritable, Text,Text, IntWritable>{
    // 输出的k
    Text k = new Text();
    // 输出的v
    IntWritable v = new IntWritable(1);

    /**
     * @param key 输入的k，
     * @param value 输入的v，这个实际就是每次进来的数据
     * @param context mapper运行时的调度
     */
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        // 将输入的数据转化成String类型，因为我们要对数据切片
        String value_content = value.toString();
        // 使用空格切分数据
        String[] words = value_content.split(" ");
        // 迭代数组
        for (String word : words) {
            k.set(word);
            context.write(k,v);
        }
        // 至此 Mapper完成

    }
}
```

- Reducer代码示例
```
/**
 * 1)继承Reducer类
 * 2)Reducer的输入类型对应Mapper的输出数据类型
 * 3)实现reduce()方法，也就是编写业务逻辑
 * 4)ReduceTask进程每一组相同k的k-v组调用一次reduce方法
 */
public class WordCountReducer extends Reducer<Text, IntWritable,Text,IntWritable> {
    IntWritable v = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        int sum = 0;
        for (IntWritable value : values) {
            sum += value.get();
        }
        v.set(sum);
        context.write(key,v);
    }
}
```

- Driver示例代码
```
public class WordCountDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        // 定义配置对象，配置相关参数
        Configuration conf = new Configuration();
        /*
        示例
        // namenode的访问地址
        conf.set("fs.defaultFS","hdfs://hadoop10:9820");
        // yarn配置
        conf.set("mapreduce.framework.name","yarn");
        // resourceManager配置，我的resourceManger在hadoop12机器上
        conf.set("yarn.resourcemanager.hostname","hadoop12");
        // 允许mr远程运行在集群上
        conf.set("mapreduce.app-submission.cross-platform","true");
        */

        // 1.创建job任务
        Job job = Job.getInstance(conf);

        // 2.设置jar包
        job.setJarByClass(WordCountDriver.class);

        // 3.关联Mapper和Reducer
        job.setMapperClass(WordCountMap.class);
        job.setReducerClass(WordCountReducer.class);

        // 4.设置Map和Reduce的输出类型
        job.setMapOutputValueClass(IntWritable.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        //5. 设置输入输出文件
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        //6.提交任务
        boolean b = job.waitForCompletion(true);

        System.exit(b ? 0:1);

    }
}
```
可以打成jar包 Hadoop_Demo1-1.0-SNAPSHOT.jar，上传到集群中运行我们wordcount程序
示例
```
[v2admin@hadoop10 ~]$ hadoop jar Hadoop_Demo1-1.0-SNAPSHOT.jar cn.leaf.demo04.WordCountDriver /home/input /home/output2
```

