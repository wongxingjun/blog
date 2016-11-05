title: "Spark Streaming学习笔记（六）"
date: 2015-03-31 09:38:27
category: Spark Streaming
tags: [Spark Streaming, 源码]
---

前面讲过Job的提交过程，但没有涉及到Job的生成和更多的调度细节，接着上源码。
<!--more-->

# Job的生成
前面已经讲到过graph.generateJobs(time)来生成Job，追踪源码
```scala
  def generateJobs(time: Time): Seq[Job] = {
    logDebug("Generating jobs for time " + time)
    val jobs = this.synchronized {
      outputStreams.flatMap(outputStream => outputStream.generateJob(time))
    }
    logDebug("Generated " + jobs.length + " jobs for time " + time)
    jobs
  }
```
这里其实outputStreams其实是一个ArrayBuffer[DStream[_]]()实例，也就是一个DStream对象吧，调用了generateJob，继续看这个

```scala
  //生成给定时间戳的Spark Streaming Job，是一个需要直接调用的内部方法。默认实例化对应的RDD。
  private[streaming] def generateJob(time: Time): Option[Job] = {
    getOrCompute(time) match {//取回或者计算对应time的RDD
      case Some(rdd) => {
        val jobFunc = () => {
          val emptyFunc = { (iterator: Iterator[T]) => {} }
          context.sparkContext.runJob(rdd, emptyFunc)//暴露Streaming的本质了，其实还是Spark核心的调用
        }
        Some(new Job(time, jobFunc))
      }
      case None => None
    }
  }
 ```
 可以看到，最终还是调用了Spark核心方法，看看这个runJob

 ```scala
  //在一个RDD分区集合上执行一个方法并返回结果到指定的处理方法。这是所有Spark中action的主入口
  def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      allowLocal: Boolean,
      resultHandler: (Int, U) => Unit) {
    if (dagScheduler == null) {
      throw new SparkException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    val start = System.nanoTime
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, allowLocal,
      resultHandler, localProperties.get)//dagScheduler来执行runJob
    ......
  }
 ```
 这里可以看到是dagScheduler来runJob，继续看这个

```scala
   def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      allowLocal: Boolean,
      resultHandler: (Int, U) => Unit,
      properties: Properties = null)
    {
    //提交job并等待结果
    val waiter = submitJob(rdd, func, partitions, callSite, allowLocal, resultHandler, properties)
    waiter.awaitResult() match {
      ......
    }
  }
```
下面自然要看看这个submitJob了

```scala
  //向JobScheduler提交一个Job并返回一个JobWaiter对象，JobWaiter直到job执行完都对这个block可用，
  //或者也可以用来cancel这个job
  def submitJob[T, U](
 		......
 		): JobWaiter[U] =
    {
    //检查这个RDD分区是否存在，操作是否合法，代码省略
    ......

    assert(partitions.size > 0)
    val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
    val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
    eventProcessActor ! JobSubmitted(    //向eventProcessActor发送JobSubmitted消息
      jobId, rdd, func2, partitions.toArray, allowLocal, callSite, waiter, properties)
    waiter
  }
 ```

跟踪eventProcessActor处理JobSubmitted的方法
 ```scala
   private[scheduler] def handleJobSubmitted(jobId: Int,  
   ......
   
  {
    var finalStage: Stage = null
    ......
    
    if (finalStage != null) {
      //创建新的Job
      val job = new ActiveJob(jobId, finalStage, func, partitions, callSite, listener, properties)
			......
			
      val shouldRunLocally =
        localExecutionEnabled && allowLocal && finalStage.parents.isEmpty && partitions.length == 1
      if (shouldRunLocally) {
        // Compute very short actions like first() or take() with no parent stages locally.
        listenerBus.post(SparkListenerJobStart(job.jobId, Array[Int](), properties))
        runLocally(job)//如果需要本地执行，go ahead
      } else {
        jobIdToActiveJob(jobId) = job
        activeJobs += job
        finalStage.resultOfJob = Some(job)
        listenerBus.post(SparkListenerJobStart(job.jobId, jobIdToStageIds(jobId).toArray,
          properties))
        submitStage(finalStage)//否则就提交stage
      }
    }
    submitWaitingStages()
  }
 ```

# 小结
可以看到，Streaming的本质也就是将Spark的一些基本RDD操作封装，其主要的调度工作还是由Spark核心调度器来完成。以上是关于Streaming的Job如何生成，生成Job之后将会划分Stage，生成task并最终调度到各个节点上运行，关于Streaming暂时分析到这里，日后补充Spark核心调度。