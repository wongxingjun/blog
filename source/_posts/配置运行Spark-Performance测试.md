title: "配置运行Spark Performance测试"
date: 2015-04-19 13:41:58
category: Spark
tags: [性能, Spark]
---

因为学习需要测试Spark Streaming的性能，在真实的生产环境中Spark Streaming会结合Kafka、Flume来部署使用。由于条件限制，无法获取类似生产环境中的数据流，因此只能采用程序自己产生数据的方式来模拟数据流。Spark开源至今，已经有了一些benchmark可以用来测试性能，但对Streaming目前还没有一些成熟的benchmark出现。偶然看到DataBricks在GitHub上开源了一个[Spark Performance测试程序](https://github.com/databricks/spark-perf)，便拿来一试。这里记录一下整个配置过程，作为笔记。
<!--more-->

# 准备工作
在这之前，要搭建一个Spark集群。我采用虚拟机搭建一个简单的虚拟集群，一共3个Xen虚拟机，vm1、vm2、vm3，其中vm1作为master，其余是slave。服务器配置8核CPU，24G内存。每个虚拟机配置4个VCPU，4G内存。其他集群配置非本文重点，不再赘述。

# 开始配置
## 下载代码
直接从GitHub上下载代码：
```
git clone git@github.com:databricks/spark-perf.git
```

## 修改配置

### 建立配置文件
根据GitHub中的描述，我们开始对测试程序进行配置。首先进入config文件夹，根据配置模板建立新的配置文件：
```
cp config.py.template config.py
```


### 修改config.py文件
- 设置根目录，`SPARK_HOME_DIR = "/root/spark-1.1.0-bin-hadoop2.4"`
- 设置集群url，`SPARK_CLUSTER_URL = "spark://192.168.226.101:7077"`
- 修改全局Java配置，spark.executor.memory，`JavaOptionSet("spark.executor.memory", ["2g"])`
- 设置spark.driver.memory，`SPARK_DRIVER_MEMORY = "512m"`
- 设置Spark Streaming选项，设置spark.executor.memory，我设置为512M

```
STREAMING_COMMON_JAVA_OPTS = [
...
JavaOptionSet("spark.executor.memory", ["512m"]),
...
]
```

- 设置要进行的测试，这个测试涵盖了Spark、MLib和Streaming测试，这里默认将前两个测试的入口注释掉，只默认开启Streaming，在line210可以看到`STREAMING_TESTS =[]`，后面添加了`basic`，`state-by-key`，`group-by-key-and-window`，`reduce-by-key-and-window`和`hdfs-recovery`共5个测试到`STREAMING_TESTS`中。

# 开始运行测试
## 准备工作
在spark-perf文件夹下，`./bin/run`，这里将会出现一个问题，因为我所有的虚拟机都在局域网中，如果直接运行，脚本会自动使用sbt编译源码，需要从外网下载一些依赖文件，这样会导致编译失败。我想到的办法是将程序在外网主机上编译后拷贝过来。将spark-perf拷贝到主机上，在streaming-tests下，执行

```
./sbt/sbt update
./sbt/sbt compile
./sbt/sbt package
```
然后还要注意一点，要在运行程序的python脚本中，将编译部分注释掉。在spark-perf/lib/sparkperf下的main.py中
```
# Build the tests for each project.
...#将这其中部分，全部注释掉
...#不然再次运行，又会去编译
# Start our Spark cluster.
```

## 开始运行
在spark-perf下
```
./bin/run
```
所有测试结果将会存到spark/perf/results文件夹下，比如`state-by-key`，会产生两个文件
state-by-key.err和state-by-key.out中，state-by-key.out为输出结果：
```
Result: count: 30, avg: 0.846 s, stdev: 0.244 s, min: 0.630 s, 
25%: 0.707 s, 50%: 0.738 s, 75%: 0.824 s, max: 1.394 s
```
