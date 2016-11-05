title: "Spark作业调度日志获取与分析"
date: 2015-05-26 16:14:43
categories: Spark
tags: [Spark, 调度]
---

Spark在Standalone模式下，作业调度采用中心调度方式。按Job-> Stage-> TaskSet-> run on executor步骤来实现作业的调度执行，因为折腾自己的一点东西需要获取Spark作业调度的详细信息，这里把获取方式和对日志的分析记录一下。
<!--more-->

目前Spark提供的Web UI界面已经可以获取很多调度细节信息，但由于我要做离线静态分析，需要把调度信息都捕获下来，想了半天，经过师兄的点播，初步想到以下方法：
- 开启Spark自带的日志打印，把日志打印到文件中，分析文件
- 源码修改UI的实现，调整页面内容，自己DIY想要的内容

鉴于目前功力还不够，我采用开启详细日志打印的配置选项，在SPARK_HOME/conf/spark-default.conf下，配置：
```
spark.eventLog.enabled           true
spark.eventLog.dir               file:///root/spark-1.1.0-bin-hadoop2.4/eventLog
```
将详细日志打印到/root/spark-1.1.0-bin-hadoop2.4/eventLog目录下。

采用python来处理eventLog，eventLog会记录详细的调度信息到一个EVENT_LOG_1文件中，以下是其中2行：
```python
{"Event":"SparkListenerApplicationStart","App Name":"BenchMark","Timestamp":1432818103984,"User":"root"}
{"Event":"SparkListenerJobStart","Job ID":0,"Stage IDs":[0,1]}
```

获取日志后，就可以对其进行静态文本分析了，我写了一个简单的python脚本，可以分析出节点被调度的比例。
```python
TaskSheduling: 
{'server5': 263, 'server6': 280, 'server7': 293, 'vm4': 8171, 'vm2': 8170, 'vm3': 8161, 'vm1': 0}
Proportion: 
{'server5': 0.0104, 'server6': 0.0111, 'server7': 0.0116, 'vm4': 0.3225, 'vm2': 0.3224, 'vm3': 0.3221, 'vm1': 0.0}
```