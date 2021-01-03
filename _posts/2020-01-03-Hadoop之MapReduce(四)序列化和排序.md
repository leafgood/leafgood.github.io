---
layout: post
title: '08Hadoop之MapReduce(四)序列化和排序'
date: 2020-01-03
author: leafgood
color: rgb(255,210,32)
tags: Hadoop入门与进阶
---
# 1.Hadoop序列化和自定义实现序列化
序列化就是把内存中的对象，转换成字节序列，或这是其他传入协议，然后进行网络传输或者持久化到磁盘中。
反序列化就是一个相反的过程，将收到的字节序列或者磁盘中持久化的数据，转换成内存中的对象。

hadoop有一套自己的序列化机制，像前面提到的BooleanWritable、ByteWritable、IntWritable、FloatWritable、LongWritable、DoubleWritable、Text、MapWritable、ArrayWritable，这些都是hadoop实现好的序列化类型，我们可以直接拿来使用。

但有些时候，这些基本的序列化类型不能满足我们的需求，我们就需要自己去实现一个序列化类型。

该怎么去实现？先看看hadoop中实现方式，就看IntWritable
```
public class IntWritable implements WritableComparable<IntWritable> 
```
它实现WritableComparable这接口，那我们自定义的类也可以实现这个接口，那么先看看这个接口有哪些要实现的方法
```
public interface WritableComparable<T> extends Writable, Comparable<T> {
}
```
好尴尬，这个接口啥都没有啊，那继续看它继承的接口，有两个。
第一个Writable接口
```
public interface Writable {
    // 序列化方法
    void write(DataOutput var1) throws IOException;
    // 反序列化方法
    void readFields(DataInput var1) throws IOException;
}
```
两个要实现的方法，一个序列化方法和一个反序列化方法

然后我们第二个接口
```
public interface Comparable<T> {
    public int compareTo(T o);
}
```
为什么要实现这个接口？

在MapReduce的shuffle过程要求key必须能够排序，所以如果自定义序列化类需要放在key中传输，那么就要实现这个接口中的方法，当然如果不需要，就可以略过了。

示例，实现一个可以序列化的StuInfoBean
```
public class StuInfoBean implements WritableComparable<StuInfoBean> {
    private Integer stuid;
    private String name;
    private Integer age;
    private Boolean sex;

    public StuInfoBean(){}

    public Integer getStuid() {
        return stuid;
    }

    public void setStuid(Integer stuid) {
        this.stuid = stuid;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public Boolean getSex() {
        return sex;
    }

    public void setSex(Boolean sex) {
        this.sex = sex;
    }

    @Override
    public String toString() {
        return "StuInfoBean{" +
                "stuid=" + stuid +
                ", name='" + name + '\'' +
                ", age=" + age +
                ", sex=" + sex +
                '}';
    }

    /**
     * 如果需要就实现，不需要就略过
     */
    @Override
    public int compareTo(StuInfoBean o) {
        return this.stuid.compareTo(o.getStuid());
    }

    @Override
    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeInt(stuid);
        dataOutput.writeUTF(name);
        dataOutput.writeInt(age);
        dataOutput.writeBoolean(sex);
    }

    /**
     * 这里要注意，反序列的顺序要和序列化保持一致
     */
    @Override
    public void readFields(DataInput dataInput) throws IOException {
        stuid = dataInput.readInt();
        name = dataInput.readUTF();
        age = dataInput.readInt();
        sex = dataInput.readBoolean();
    }
}
```

# 2.排序
MapTask收集我们的map()方法输出的kv对，放到内存缓冲区中，这是一个环形数据结构，其实就是一个字节数组，叫kvbuffer。
里面存放着我们的数据及索引数据kvmeta。
kvbuffer的大小虽然可以设置，但终归是有限的，当写满的时候，内存中的数据就会刷到磁盘上，这个过程就是溢写，溢写产生的文件可能会不止一个，多个溢出文件会被合并成大的溢出文件。
在这个过程中，都要调用Partitioner进行分区和针对key进行排序sort。
分区之前已经说了，这里就说下排序sort。

MapTask和ReduceTask均会对数据按照key进行排序，任何应用程序中的数据不管业务逻辑是否需要，在hadoop中均会被排序。

默认是按照字典排序，实现方式是快排。

自定义类型排序的实现，就是前面compareTo方法。
```
  @Override
    public int compareTo(StuInfoBean o) {
        return this.stuid.compareTo(o.getStuid());
    }
```

# 3.规约（Combiner）
这个也是合并，每一个 map 都可能会产生大量的本地输出，Combiner 的作用就是对 map 端的输出先做一次合并，以减少在map 和 reduce节点之间的数据传输量，以提高网络IO性能，是MapReduce 的一种优化手段之一。
但有个前提就是不能改变业务最终逻辑，Combiner的输出kv应该跟Reducer的输入kv类型要对应起来。

Combiner是MR程序中Mapper和Reducer之外的一种组件，它的父类就是Reduce。

举个例子
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
需要求出每个属性值的和，那么我们可以自定义MyCombiner，先局部求和，最后汇总到Reduce求和。

mapper代码
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

MyCombiner代码
```
public class MyCombiner extends Reducer<Text, IntWritable,Text,IntWritable> {
    private IntWritable v= new IntWritable();
    
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        int sum = 0;
        for (IntWritable value : values) {
            sum+=value.get();
        }
        v.set(sum);
        context.write(key,v);
    }
}
```
Reduce代码
```
public class ClReduce extends Reducer<Text, IntWritable,Text,IntWritable> {
//    private Text outk = new Text();
    private IntWritable outv = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        for (IntWritable value : values) {
            outv.set(value.get());
        }
        context.write(key,outv);
    }
}
```

Driver代码
```
public class ClDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Configuration conf = new Configuration();

        Job job = Job.getInstance(conf);

        job.setJarByClass(ClDriver.class);

        job.setMapperClass(ClMap.class);
        job.setReducerClass(ClReduce.class);

        job.setMapOutputValueClass(IntWritable.class);
        
        // 设置使用MyCombiner
        job.setCombinerClass(MyCombiner.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        boolean b = job.waitForCompletion(true);

        System.exit(b ? 0:1);

    }
}
```

























