title: "Spark Streaming学习笔记（三）"
date: 2015-03-25 16:18:00
category: Spark Streaming
tags: [Spark Streaming, 源码]
---

接着上一篇来分析一个典型streaming应用背后的启动过程。
<!--more-->

# 源码跟踪
StreamingContext中声明了一个JobScheduler对象：
```scala
private[streaming] val scheduler = new JobScheduler(this)
```
由此追踪JobScheduler中的start方法
``` scala
def start(): Unit = synchronized {
    if (eventActor != null) return // scheduler has already been started

    logDebug("Starting JobScheduler")
    eventActor = ssc.env.actorSystem.actorOf(Props(new Actor {
      def receive = {
        case event: JobSchedulerEvent => processEvent(event)//处理JobSchedulerEvent事件
      }
    }), "JobScheduler")

    listenerBus.start()//启动StreamingListenerBus监听器
    receiverTracker = new ReceiverTracker(ssc)
    receiverTracker.start()//启动ReceiverTracker 
    jobGenerator.start()//启动JobGenerator
    logInfo("Started JobScheduler")
 }
```

`listenerBus`是一个StreamingListenerBus对象，用来异步向StreamingListener传递StreamingListenerEvents
下面看一下ReceiverTracker中的start方法
```scala
 /** Start the actor and receiver execution thread. */
def start() = synchronized {
    if (actor != null) {
      throw new SparkException("ReceiverTracker already started")
    }

    if (!receiverInputStreams.isEmpty) {//receiverInputStreams不能为空
      actor = ssc.env.actorSystem.actorOf(Props(new ReceiverTrackerActor),
        "ReceiverTracker")//实例化ReceiverTrackerActor
      receiverExecutor.start()//启动receiver
      logInfo("ReceiverTracker started")
    }
}
```
receiverExecutor是一个receiverLauncher对象，receiverLauncher线程类执行集群上所有的receiver。重点关注一下其中一个startReceiver方法：
```scala
 /**
  * Get the receivers from the ReceiverInputDStreams, distributes them to the
  * worker nodes as a parallel collection, and runs them.
  */
 private def startReceivers() {
        val receivers = receiverInputStreams.map(nis => {
        val rcvr = nis.getReceiver()
        rcvr.setReceiverId(nis.id)
        rcvr
      })

      // Right now, we only honor preferences if all receivers have them
      val hasLocationPreferences = receivers.map(_.preferredLocation.isDefined).reduce(_ && _)

      //创建一个receiver的并行集合临时RDD，分发到各个worker节点上
      val tempRDD =
        if (hasLocationPreferences) {
          val receiversWithPreferences = receivers.map(r => (r, Seq(r.preferredLocation.get)))
          ssc.sc.makeRDD[Receiver[_]](receiversWithPreferences)
        } else {
          ssc.sc.makeRDD(receivers, receivers.size)
        }

      //启动woker上的receiver的方法（这是方法式的神奇~~）
      val startReceiver = (iterator: Iterator[Receiver[_]]) => {
        if (!iterator.hasNext) {
          throw new SparkException(
            "Could not start receiver as object not found.")
        }
        val receiver = iterator.next()
        val executor = new ReceiverSupervisorImpl(receiver, SparkEnv.get)
        executor.start()//启动这一个receiver，详情请看下面
        executor.awaitTermination()
      }
     //运行这个简单作业来确定所有的woker都注册过，避免将所有receiver分到一个节点上
      if (!ssc.sparkContext.isLocal) {
        ssc.sparkContext.makeRDD(1 to 50, 50).map(x => (x, 1)).reduceByKey(_ + _, 20).collect()
      }

      // 分发receiver并启动之
      logInfo("Starting " + receivers.length + " receivers")
      ssc.sparkContext.runJob(tempRDD, startReceiver)//这里也可以看出streaming最终调用spark核心
      logInfo("All of the receivers have been terminated")
 }
```

分发receiver的时候runJob带了一个方法参数，startReceiver，这个方法是一个内部定义的（上面代码中有注释说明），这个方法实例化了一个ReceiverSupervisorImpl对象，也就是将会执行的executor，并调用了start方法启动，很明显，这个start方法定义在了ReceiverSupervisorImpl中
```scala
/** Start the supervisor */
  def start() {
    onStart()
    startReceiver()
  }
```

继续看最终可以追溯到一个onStart方法，和一个startReceiver方法，onStart方法在ReceiverSupervisorImpl中有实现：
```scala
  override protected def onStart() {
    blockGenerator.start()//开启blockGenerator开始生成数据块，这里的细节暂时不讨论
  }
```

startReceiver方法在ReceiverSupervisor中实现：
```scala
  /** Start receiver */
  def startReceiver(): Unit = synchronized {
    try {
      logInfo("Starting receiver")
      receiver.onStart()
      logInfo("Called receiver onStart")
      onReceiverStart()
      receiverState = Started
    } catch {
      case t: Throwable =>
        stop("Error starting receiver " + streamId, Some(t))
    }
  }
```

这里的receiver.onStart()是启动一个SocketReveiver对象receiver，在SocketInputDStream类中真正其实现如下：
```scala
 def onStart() {
    // Start the thread that receives data over a connection
    new Thread("Socket Receiver") {
      setDaemon(true)
      override def run() { receive() }
    }.start()
  }
```

启动一个receive线程，receive()的实现紧接着在onStart()下实现：
```scala
  /** Create a socket connection and receive data until receiver is stopped */
  def receive() {
    var socket: Socket = null
    try {
      logInfo("Connecting to " + host + ":" + port)
      socket = new Socket(host, port)
      logInfo("Connected to " + host + ":" + port)
      val iterator = bytesToObjects(socket.getInputStream())
      while(!isStopped && iterator.hasNext) {
        store(iterator.next)
      }
      logInfo("Stopped receiving")
      restart("Retrying connecting to " + host + ":" + port)
    } catch {
      case e: java.net.ConnectException =>
        restart("Error connecting to " + host + ":" + port, e)
      case t: Throwable =>
        restart("Error receiving data", t)
    } finally {
      if (socket != null) {
        socket.close()
        logInfo("Closed socket to " + host + ":" + port)
      }
    }
  }
}
```

可以看出在这里是真正创建一个socket并实现连接，接收数据。

# 小结
以上是wordCount启动流程，并没有涉及到本质的细节，接下来将会详细探讨一下
- receiver如何接收和保存流数据
- sparkContext中runJob执行调度细节

