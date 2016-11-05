title: "Spark是如何工作的"
date: 2015-05-11 20:53:36
categories: Spark
tags: Spark
---

Databricks的Reynold Xin在Quora上对[How does Apache Spark work?](https://www.quora.com/How-does-Apache-Spark-work)的回答。觉得比较精辟，所以这里将其翻译了一下。
<!--more-->
从多个角度来讲，Spark是对MapReduce范式的一个更好的实现（Spark的缔造者Matei早年也从事很多关于Hadoop MapReduce的工作，大家对此不要感到很惊讶）

从编程抽象的角度看：
---
- Spark概括了Map/Reduce两个stage来支持复杂的DAG（Directed Acyclic Graph，有向无环图）任务集。大多数计算都映射到了带有依赖关系的map和reduce中。Spark的RDD编程模型为这些依赖关系建立DAG模型，这样就可以很自然地表达计算了。

- 通过更好的语言集成来为数据流建立模型，Spark可以不需要Hadoop MapReduce中的大量样板代码。尤其是当你看一个Hadoop MapReduce的程序时，因为大量的样版代码干扰，你很难知道程序要做什么，但阅读Spark程序却很自然。

- 因为使用了更多灵活的编程模型，一些在Hadoop MapReuce中嵌入内部的关键操作在Spark中都被放在了“应用空间”。这意味着应用程序可以重写shuffle和aggregation的实现方式，这在MapReduce中是不可能的。一般很多应用程序不这么做，但是这样可以使得一些特定的应用可以为特定的负载做优化。

- 应用程序可以选择将数据集存在一个集群的内存中。这种在很多应用中实际上是很基本的简单原始的方式在Spark中需要多次扫描中间数据集（比如说很多机器学习算法都是迭代的）

从更高层的系统工程师角度看：
---
- Spark用快的RPC来发送和调度任务。

- Spark使用线程池来执行任务，而不是JVM的进程池。

    **以上两点结合起来可以使得Spark的调度时延达到ms级别，然而MapReduce一般要几秒，在一些很忙的集群中甚至要数分钟。**

- Spark可以同时支持基于检查点（也就是MapReduce中的容错方式）和基于血统图的恢复机制。基于血统图的错误恢复一般要快一些（因为对高数据流吞吐应用来说，通过网络复制状态是很慢的），除非错误是很常见的。

- 可能是由于其根本学术目的，Spark社区经常有一些新奇的想法。比如使用torrent-like协议来实现一对多的数据广播。