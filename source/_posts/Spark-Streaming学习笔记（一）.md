title: "Spark Streaming学习笔记（一）"
date: 2015-03-23 15:46:20
category: Spark Streaming
tags: [Spark Streaming, 源码]
---

Spark Streaming是构建在Spark核心引擎上的实时流处理框架，吞吐率远超已有的实时流处理框架。由于项目中要涉及到Streaming的相关知识，我会陆续将一些Spark Streaming的学习笔记整理出来放到博客中以做留存。
<!--more-->

# 实例分析
下面看一个源码给出的例子程序：
``` scala
package org.apache.spark.examples.streaming
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.streaming.StreamingContext._
import org.apache.spark.storage.StorageLevel

object NetworkWordCount {
  def main(args: Array[String]) {
    if (args.length < 2) {
      System.err.println("Usage: NetworkWordCount <hostname> <port>")
      System.exit(1)
    }
    StreamingExamples.setStreamingLogLevels()
    val sparkConf = new SparkConf().setAppName("NetworkWordCount")
    val ssc = new StreamingContext(sparkConf, Seconds(1))
    val lines = ssc.socketTextStream(args(0), args(1).toInt, 
                                     StorageLevel.MEMORY_AND_DISK_SER)
    val words = lines.flatMap(_.split(" "))
    val wordCounts = words.map(x => (x, 1)).reduceByKey(_ + _)
    wordCounts.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```
这个例子最基本的功能就是从一个socket不断读取数据流，以1S为单位统计word个数并打印结果。下面我们看一下程序的整体流程：

## 创建SparkConf
```scala
val sparkConf = new SparkConf().setAppName("NetworkWordCount")
```
创建一个新的SparkConf对象，并将应用名设置成NerworkWordCount

## 创建Spark Streaming Context
```scala
val ssc = new StreamingContext(sparkConf, Seconds(1))
```
根据sparkConf创建一个StreamingContext，StreamingContext继承自SparkContext，是Streaming程序的入口。

## 从socket读取数据流
```scala
val lines = ssc.socketTextStream(args(0), args(1).toInt, 
                                 StorageLevel.MEMORY_AND_DISK_SER)
```
arg(0)是IP地址，args(1)是端口号。调用socketTextStream从socket读取文本数据流，并处理成行。

## 处理数据流
```scala
val words = lines.flatMap(_.split(" "))
```
flatMap是DStream的一种转换操作，我们先简单理解为，将一行行文本以空格来分割，处理成单个的word。
```scala
val wordCounts = words.map(x => (x, 1)).reduceByKey(_ + _)
```
开始进行wordcount。
```scala
wordCounts.print()
```
打印wordcount计算结果。

## 启动程序
```scala
 ssc.start()
 ssc.awaitTermination()
```
调用start方法正式开始进行流处理计算，awaitTermination指等待终止，真实环境中流处理是不间断进行的，因此这里采用这种方式，并不是直接stop。
