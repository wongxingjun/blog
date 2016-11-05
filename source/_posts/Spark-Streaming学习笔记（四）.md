title: "Spark Streaming学习笔记（四）"
date: 2015-03-26 17:01:44
category: Spark Streaming
tags: [Spark Streaming, 源码]
---

上文讲到了wordCount示例程序运行时的启动流程，依旧停留在框架理解上。下面开始详细介绍一下Spark Streaming中是如何接收和存储流数据。实际应用中Streaming的输入源有多种，这里仍旧以wordCount为例，对socketStream进行介绍。
<!--more-->

# 源码跟踪
上文中StartReceivers中实例化了一个ReceiverSupervisorImpl对象，然后启动之。ReceiverSupervisorImpl继承自ReceiverSupervisor，实现了ReceiverSupervisor中处理接收数据的方法。这里executor.start()中的start方法是继承自ReceiverSupervisor，调用了onStart和startReceiver两个方法。继续看startReceiver方法：
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
终于看到启动receiver的关键代码了，可是onStart和onReceiverStart方法其实是在ReceiverSupervisorImpl类中实现的，我们看看代码：

```scala
override protected def onStart() {
    blockGenerator.start()//blockGenerator是一个BlockGenerator对象，顾名思义就是数据块生成器
}
```
```scala
override protected def onReceiverStart() {
    val msg = RegisterReceiver(
      streamId, receiver.getClass.getSimpleName, Utils.localHostName(), actor)
    val future = trackerActor.ask(msg)(askTimeout)//发送消息给trackerActor，通知receiver启动
    Await.result(future, askTimeout)
  }
```
接着看BlockGenerator中的start方法：

```scala
/** Start block generating and pushing threads. */
def start() {
    blockIntervalTimer.start()//启动blockInterval定时器
    blockPushingThread.start()//启动push block线程
    logInfo("Started BlockGenerator")
  }
```
接下来，看看blockPushingThread乃何方神圣:

```scala
private val blockPushingThread = new Thread() { override def run() { keepPushingBlocks() } }
```
调用了一个叫keepPushingBlocks的方法，继续往根上刨：

```scala
/** Keep pushing blocks to the BlockManager. */
private def keepPushingBlocks() {
    logInfo("Started block pushing thread")
    try {
      while(!stopped) {//不断将blocksForPushing中的block取出来push之
        Option(blocksForPushing.poll(100, TimeUnit.MILLISECONDS)) match {
          case Some(block) => pushBlock(block)//调用pushBlock
          case None =>
        }
      }
      // Push out the blocks that are still left
     ......
 }
```
接着看关键的pushBlock方法，pushBlock调用onPushBlock方法

```scala
def onPushBlock(blockId: StreamBlockId, arrayBuffer: ArrayBuffer[_])//当有新数据块需要push便调用
```
这里并没有具体实现，具体实现其实在ReceiverSupervisorImpl中实例化BlockGenerator时：

```scala
def onPushBlock(blockId: StreamBlockId, arrayBuffer: ArrayBuffer[_]) {
      pushArrayBuffer(arrayBuffer, None, Some(blockId))
}
```
由此，继续刨根问底，看pushArrayBuffer方法：

```scala
/** Store an ArrayBuffer of received data as a data block into Spark's memory. */
def pushArrayBuffer(
      arrayBuffer: ArrayBuffer[_],
      optionalMetadata: Option[Any],
      optionalBlockId: Option[StreamBlockId]
    ) {
    val blockId = optionalBlockId.getOrElse(nextBlockId)
    val time = System.currentTimeMillis
    blockManager.putArray(blockId, arrayBuffer.toArray[Any], storageLevel, tellMaster = true)//将block保存到blockManager中
    logDebug("Pushed block " + blockId + " in " + (System.currentTimeMillis - time)  + " ms")
    reportPushedBlock(blockId, arrayBuffer.size, optionalMetadata)//向driver发送block的id，大小和元数据信息
  }
```
至此，流数据接收并存储起来，然后将block的id，大小和元数据信息发送给dirver以供调度时使用。

# 膜拜大神
下图是大神[徽沪一郎](http://www.cnblogs.com/hseagle/) [博客](http://www.cnblogs.com/hseagle/p/3673139.html)中给出的一幅Spark Streaming接收数据示意图
![receiveStream](/figures/spark-learning/receiveStream.png)
一目了然，膜拜一下~~~
