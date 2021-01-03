---
layout: post
title: '07Hadoop之MapReduce(三)Shuffle机制和Partition分区'
date: 2020-01-03
author: leafgood
color: rgb(255,210,32)
tags: Hadoop入门与进阶
---
## 1.Shuffle机制
Shuffle指的map()方法之后，reduce()之前的数据处理过程。

就是将 MapTask 输出的结果数据，按照 Partitioner 分区制定的规则分发给ReduceTask执行，并在分发的过程中，对数据进行分区和排序。


## 2.Partition分区
在进行MapReduce计算时，有时候需要把最终的输出数据分到不同的文件中，这时候，就需要用到分区。

举个例子：
有这样一组数据
```
class1_aaa 50
class2_bbb 100
class3_ccc 80
class1_ddd 10
class2_eee 100
class3_fff 70
class1_hhh 150
class2_lll 100
class3_www 80
```
想实现这样一个需求，对里面的数据的值分组求和，并将结果分别输出一个文件，那这里就需要Partition分区了。

Map 代码示例
```
public class ClMap extends Mapper<LongWritable, Text,Text, IntWritable> {
    // 输出的k和v
    Text outk = new Text();
    IntWritable outv = new IntWritable();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] contents = line.split(" ");
        String outkey = contents[0].split("_")[0];

        outk.set(outkey);
        outv.set(Integer.parseInt(contents[1]));

        context.write(outk,outv);
    }
}
```

Partition代码示例
```
public class ClPartitioner extends Partitioner<Text, IntWritable> {
    @Override
    public int getPartition(Text text, IntWritable intWritable, int numPartitions) {
        String strStarts = text.toString().split("_")[0];

        if("class1".equals(strStarts)){
            return 0;
        }else if("class2".equals(strStarts)){
            return 1;
        }else{
            return 2;
        }
    }
}
```

Reduce代码示例
```
public class ClReduce extends Reducer<Text, IntWritable,Text,IntWritable> {
    IntWritable outv = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        int sum = 0;
        for (IntWritable value : values) {
            sum += value.get();
        }
        outv.set(sum);
        context.write(key,outv);
    }
}
```

Driver代码示例
```
public class ClDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Configuration conf = new Configuration();

        Job job = Job.getInstance(conf);

        job.setJarByClass(ClDriver.class);

        job.setMapperClass(ClMap.class);
        job.setReducerClass(ClReduce.class);

        job.setMapOutputValueClass(IntWritable.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        // ReduceTask进程数
        job.setNumReduceTasks(3);
        // 设定使用的分区
        job.setPartitionerClass(ClPartitioner.class);

        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        
        boolean b = job.waitForCompletion(true);

        System.exit(b ? 0:1);

    }
}
```




































