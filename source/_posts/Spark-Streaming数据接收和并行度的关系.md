title: "Spark Streaming数据接收和并行度的关系"
date: 2015-06-09 15:19:37
categories: Spark Streaming
tags: [Spark Streaming, 性能]
---

Spark Streaming在运行过程中数据来源分为两种，一种是从例如hdfs的文件系统直接读取，另外就是实时从网络接收数据，其中后者需要由运行在worker节点上的receiver来完成，具体接收数据过程在源码分析中已经简要介绍。这次就receiver和系统并行度的关系进行探讨和实验。
<!--more-->

# 数据接收
Spark Streaming通过运行在worker上的executor中的receiver来接收数据，并将其存为RDD分区（内存不够会spill到磁盘）供后续计算用。如果想要接收多个InputDStream，可以设置多个receiver同时运行在多个worker node上，同时接收数据，在处理的时候可以用union转换操作来合并成一个DStream然后进行操作。receiver会消耗worker上的一个slot，因此集群的core数量要多于receiver数量，不然就只能接收数据，不能处理数据了。

# 数据接收与并行度
receiver从网络中接收数据，反序列化之后存入Spark内存中，因此所在的节点对CPU和网络资源需求高于其他计算节点（甚至有时候设置不当，内存不足，数据会溢写到磁盘中）。实际部署中有可能会遇到receiver成为整个系统的性能瓶颈。这种情况下，可以通过以下方式来适当调整：
- 设置多个receiver，并行接收数据，创建多个DStream最后union统一处理
- 设置合适的blockInterval，过小会导致block过多，最后调度时task过多，增加调度时延。过大会导致task过少，集群的core利用不足，全局并行度变差。
- 对单个receiver接收的DStream进行repartition操作，将该DStream内的RDD进行重新分区，增加分区数，可以将接收到的batch分散于集群中的其他节点上，然后再进行计算处理，增加并行度。

# 实验
用一个简单的SocketStream小程序，在两种方式下进行测试。集群中有7个节点，1个为master，其余为worker，配置2GB内存，2 cores.只用一个receiver来接收socketStream。分析日志文件，查看调度记录。
## 单个receiver
直接一个receiver接收数据，日志分析发现有近88%的调度发生在receiver所在节点。其余worker平均2%。
## 增加receiver
增加一个receiver后，两个receiver接收数据，日志分析发现task调度比例43%和42%集中在两个receiver节点，其余worker平均3%-4%。
## 重新分区
单个receiver情况下，对InputDStream进行重新分区，分为3个分区，结果发现task调度比例58%集中在receiver节点，其余worker节点平均8%-9%。

>重新分区会使用RDD.repartition()函数来进行重新分区，repartition()又进一步调用了RDD的一个转换操作函数coalesce()，这个函数可以将RDD进行重新分区。
```scala
  def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = {
    coalesce(numPartitions, shuffle = true)
  }
```
>shuffle=false时，只能减少分区，shuffle=true时，可以增加分区，将一个RDD平均划分到numPartitions个分区中，实现细节这里不讨论。shuffle操作的代价是明显的，如果设置不当，甚至会导致重新分区后Streaming的性能低于不分区的情况。

# 小结
数据接收对Spark Streaming的并行度有直接影响，这里主要关注的是receiver的设置对并行度的影响，其实各有优缺点。比如重新分区，在实验中发现重新分区的确可以提高并行度，但时延却没有降低（有时候反而增加），可能是由于虚拟机集群的缘故。

# 参考资料
[1] [Apache Spark Document](http://spark.apache.org/docs/latest/streaming-programming-guide.html#input-dstreams-and-receivers)
[2] [Apache Spark API](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.DStream)