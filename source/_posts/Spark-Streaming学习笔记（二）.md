title: "Spark Streaming学习笔记（二）"
date: 2015-03-25 10:41:58
category: Spark Streaming
tags: [Spark Streaming, 源码]
---

之前只对一个简单wordCount示例进行了简单的结构梳理，并没有涉及背后的执行原理，今天进一步了解一下背后的各种调用。
<!--more-->

# 创建SparkConf
查看源码，SparkConf是Spark核心中的一个类，用来为Spark应用设置各种参数，比如这里的AppName，其方法原型是：
```scala
def setAppName(name: String): SparkConf = {
    set("spark.app.name", name)
}
```

# 创建Streaming Context
创建一个StreamingContext，类似SparkContext，这里的StreamingContext其实是将SparkContext进行扩展调用，将其用于更加复杂的Streaming中，可以看到源码中有一个SparkContext的参数。
```scala
class StreamingContext private[streaming] (
    sc_ : SparkContext,
    cp_ : Checkpoint,
    batchDur_ : Duration
  ) extends Logging {
......
}
```

# 从socket读取数据流
socketTextStream是StreamingContext类中的一个方法
```scala
def socketTextStream(
      hostname: String,
      port: Int,
      storageLevel: StorageLevel = StorageLevel.MEMORY_AND_DISK_SER_2
      ): ReceiverInputDStream[String] = {
    socketStream[String](hostname, port, SocketReceiver.bytesToLines, storageLevel)
}
```
socketTextStream调用socketStream来读取socket中的文本数据流，socketStream也定义在StreamingContext类中，
```scala
def socketStream[T: ClassTag](
      hostname: String,
      port: Int,
      converter: (InputStream) => Iterator[T],
      storageLevel: StorageLevel
    ): ReceiverInputDStream[T] = {
    new SocketInputDStream[T](this, hostname, port, converter, storageLevel)
}
```
这里new了一个SocketInputDStream对象用来接收数据流并返回ReceiverInputDStream类型。
```scala
class SocketInputDStream[T: ClassTag](
    @transient ssc_ : StreamingContext,
    host: String,
    port: Int,
    bytesToObjects: InputStream => Iterator[T],
    storageLevel: StorageLevel
  ) extends ReceiverInputDStream[T](ssc_) {
......
}
```

# 处理数据流
这里的是对DStream的一系列的transformation，具体细节这里暂时不做详细讨论，后续会对Streaming中DStream各种操作进行介绍。只需要知道，这里就是wordCount的核心操作就可以。

# 正式启动程序
前面提到过StreamingContext是Streaming程序的入口，对整个程序的控制也在它上面进行，ssc.start()这个方法看似简单，后面大有文章。我们来看定义在StreamingContext中的start方法:
```scala
def start(): Unit = synchronized {
    // Throw exception if the context has already been started once
    // or if a stopped context is being started again
    if (state == Started) {
      throw new SparkException("StreamingContext has already been started")
    }
    if (state == Stopped) {
      throw new SparkException("StreamingContext has already been stopped")
    }
    validate()    //调用validate判断DStreamGraph是否合法
    scheduler.start()     //正式启动调度器
    state = Started
}
```
Spark Streaming的调度比较复杂，这篇文章仅仅对源码进行一个初步追踪理解，下一篇将会对调度进行讲解。
