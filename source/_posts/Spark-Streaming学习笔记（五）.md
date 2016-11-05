title: "Spark Streaming学习笔记（五）"
date: 2015-03-30 11:39:29
category: Spark Streaming
tags: [Spark Streaming, 源码]
---
接着前面的讲，这篇介绍一下Spark Streaming的调度细节，首先来看看Job的提交。
<!--more-->

# 承前启后
在StreamingContext中有一个start方法，调用schduler.start()来启动调度器（其实这里是一个实例化的JobScheduler对象）
```scala
  /**
   * Start the execution of the streams.
   */
  def start(): Unit = synchronized {
    // Throw exception if the context has already been started once
    // or if a stopped context is being started again
    if (state == Started) {
      throw new SparkException("StreamingContext has already been started")
    }
    if (state == Stopped) {
      throw new SparkException("StreamingContext has already been stopped")
    }
    validate()
    scheduler.start()//就是这句，我们以此为入口，开始刨根问底:-)
    state = Started
  }
```

# Job的提交过程
跟踪scheduler.start()方法（最好使用Intellij IDEA来看源码，可以直接快捷定位，免去了寻找的麻烦，关键是教育版免费:-)，我之前是用Sublime来看的，现在想起来，哭瞎T_T）。
```scala
  def start(): Unit = synchronized {
    if (eventActor != null) return // scheduler has already been started

    logDebug("Starting JobScheduler")
    eventActor = ssc.env.actorSystem.actorOf(Props(new Actor {
      def receive = {
        case event: JobSchedulerEvent => processEvent(event)//处理调度事件消息
      }
    }), "JobScheduler") 
    ......
    ......
    jobGenerator.start()//当当当当，主角上场了
    logInfo("Started JobScheduler")
  }
```
走进jobGenerator中的start方法

```scala  
/** Start generation of jobs */
  def start(): Unit = synchronized {
    if (eventActor != null) return // generator has already been started

    eventActor = ssc.env.actorSystem.actorOf(Props(new Actor {
      def receive = {
        case event: JobGeneratorEvent =>  processEvent(event)//处理JobGenerator事件消息
      }
    }), "JobGenerator")
    if (ssc.isCheckpointPresent) {
      restart()//如果有checkPoint，就从这里重启
    } else {
      startFirstTime()//没有checkPoint，就直接上
    }
  }
```
真正的调度发生在事件消息处理阶段，我们看processEvent

```scala
    private def processEvent(event: JobGeneratorEvent) {
    logDebug("Got event " + event)
    event match {
      case GenerateJobs(time) => generateJobs(time)
      case ClearMetadata(time) => clearMetadata(time)
      case DoCheckpoint(time) => doCheckpoint(time)
      case ClearCheckpointData(time) => clearCheckpointData(time)
    }
  }
```
那么接下来就要看generateJobs了

```scala
  /** Generate jobs and perform checkpoint for the given `time`.  */
  private def generateJobs(time: Time) {
    SparkEnv.set(ssc.env)
    Try(graph.generateJobs(time)) match {//生成time时间戳的Jobs
      case Success(jobs) =>//成功了就开始下一步的block消息获取
        val receivedBlockInfo = graph.getReceiverInputStreams.map { stream =>
          val streamId = stream.id
          val receivedBlockInfo = stream.getReceivedBlockInfo(time)
          (streamId, receivedBlockInfo)
        }.toMap
        jobScheduler.submitJobSet(JobSet(time, jobs, receivedBlockInfo))//提交JobSet
      case Failure(e) =>//错误就报告呗
        jobScheduler.reportError("Error generating jobs for time " + time, e)
    }
    eventActor ! DoCheckpoint(time)//把checkpoint消息发送到eventActor
  }
```
到了这一步，就看看提交JobSet了，submitJobSet走起~~

```scala
  def submitJobSet(jobSet: JobSet) {
    if (jobSet.jobs.isEmpty) {
      logInfo("No jobs added for time " + jobSet.time)
    } else {
      jobSets.put(jobSet.time, jobSet)//将jobSet放进JobSets的HashMap中
      jobSet.jobs.foreach(job => jobExecutor.execute(new JobHandler(job)))
                                            //开始执行这个Job并为其添加一个handler
     //查看代码这个JobHandler处理发送两个消息，一个是JobStarted，一个是JobCompleted
      logInfo("Added jobs for time " + jobSet.time)
    }
  }
```

# 小结
这里从源码开始简单介绍了一下Job的提交过程，实际上Streaming的调度最终要落实到task上，将会在下面进行分析。

