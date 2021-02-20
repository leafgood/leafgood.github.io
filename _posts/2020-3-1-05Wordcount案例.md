---
layout: post
title: '(五)集合练习--WordCount案例'
date: 2020-01-13
author: leafgood
color: rgb(98,170,255)
tags: Scala
---

# 3 集合练习-wordcount
## 3.1  需求
统计文件中单词出现的个数并进行排序，输出前三的结果

## 3.2  代码实现
示例1
```
import scala.io.Source

object Scala_WordCount {
  def main(args: Array[String]): Unit = {
     // 需求：统计文件中单词出现的个数，并进行排序，输出前三的结果
     // 1.读取文件，获取每一行的字符串
    val src = Source.fromFile("input/word.txt")
    val strList = src.getLines().toList
    src.close()

     // 2.分词
     val words = strList.flatMap(str => {
       str.split(" ")
     })

    // 3.单词进行分组，相同单词放置一个组中
    val wordGroup = words.groupBy(word => word)

    // 4.对分组后的数据进行统计
    val wordCount = wordGroup.map(kv => {
      (kv._1, kv._2.length)
    })
    // 5.排序并取前三的结果
    val top3 = wordCount.toList.sortBy(kv => kv._2).take(3)
    println(top3)
  }
}

```

代码可以进行简化
示例2
```
import scala.io.Source

object Scala_WordCount {
  def main(args: Array[String]): Unit = {
     // 需求：统计文件中单词出现的个数，并进行排序，输出前三的结果
    val src = Source.fromFile("input/word.txt")
    val strList = src.getLines().toList
    src.close()

    val top3 = strList.flatMap(str => {
       str.split(" ")
     }).groupBy(word => word)
       .map(kv => {
      (kv._1, kv._2.length)
    }).toList
       .sortBy(kv => kv._2)
       .take(3)
    println(top3)
  }
}
```