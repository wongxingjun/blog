title: Spark on YARN配置日志Web UI
date: 2016-11-07 16:38:02
categories: Spark
tags: [Spark, 配置]
---

Spark部署在YARN之后，从Standalone模式下的Spark Web UI直接无法看到执行过的application日志，不利于性能分析。得益于实验室师弟的帮忙，本文记录如何配置history UI。
<!--more-->

# 修改spark-defaults.conf
修改$SPARK_HOME/conf/spark-default.conf
```
spark.eventLog.enabled    true
spark.eventLog.compress   true
spark.eventLog.dir 		  file:///home/path/to/eventLog
spark.yarn.historyServer.address   master:18080
```
其中，spark.eventLog.enabled开启时间记录，默认是false，spark.eventLog.compress是否压缩记录Spark事件，默认snappy，spark.eventLog.dir存储日志路径，可以是hdfs路径（如hdfs://master:9000/history）或者driver本地路径，在此之前要创建相应的目录，否则会报错。spark.yarn.historyServer.address是设置Spark history server的地址和端口，这个链接将会链接到YARN监测界面上的Tracking UI。
![trackingui](/figures/spark-yarn-ui/trackingui.png)
这样点击History就可以跳转到Spark的Web UI查看相应的日志
![trackingui](/figures/spark-yarn-ui/trackingui2.png)

# 修改sparn-env.sh
```
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 -Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=file:///home/path/to/eventLog"
```
ui.port端口号需要和spark-defaults.conf保持一致，retainedApplications表示在historyServer上显示的最大application数量，如果超过这个数量，旧的application信息将会删除。fs.logDirectory日志目录和spark-defaults.conf保持一致

# 启动Spark History Server
在spark目录下
```
./sbin/start-history-server.sh
```
成功后，打开http://master:18080，就可以看到相应的日志记录列表，进去之后也可以转到Spark Web UI上
![historyServer](/figures/spark-yarn-ui/historyServer.png)