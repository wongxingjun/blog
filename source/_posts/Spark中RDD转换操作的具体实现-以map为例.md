title: "Spark中RDD转换操作的具体实现--以map为例"
date: 2015-05-21 20:38:53
categories: Spark
tags: [Spark,编程]
---

Spark使用RDD（弹性分布式数据集）来抽象分布式内存，提供一种抽象数据类型，具有高度容错的特点，对编程者提供了一种极为便捷的数据操作方式。Spark内部提供了丰富的RDD转换操作为编程者提供了丰富的RDD转换接口。学习Spark编程的第一步就是要了解一些常见的转换操作，最近看了一些常见的Spark中RDD转换操作，这里以最简单的map操作来学习一下Spark内部到底是如何进行RDD转换计算的。
<!--more-->

RDD的产生可以有两种途径：从已有的文件或者内部集合直接创建RDD，或者从现有的RDD通过施加一些转换操作转换得来。map是RDD转换操作中最简单的一种，和MapReduce中的map功能基本一样，就是将数据映射成元组集合，在源码中定义为：
```scala
def map[U: ClassTag](f: T => U): RDD[U] = new MappedRDD(this, sc.clean(f))
```

将RDD中类型为T的元素一一映射成类型为U的元素，我们可以看到这里map函数就是实例化了一个MappedRDD，顾名思义，这就是map之后的RDD类型，MappedRDD自然是继承RDD基类了。我们看MappedRDD的定义：
```scala
private[spark]
class MappedRDD[U: ClassTag, T: ClassTag](prev: RDD[T], f: T => U)
  extends RDD[U](prev) {

  override def getPartitions: Array[Partition] = firstParent[T].partitions//获取firstParent分区信息

  override def compute(split: Partition, context: TaskContext) =
    firstParent[T].iterator(split, context).map(f)//这里才是真正的计算步骤
  }
```
firstParent是当前要map得到的RDD（其实也就是mappedRDD）的第一个父RDD，这里调用父RDD的iterator方法获取iter对象，iterator方法在RDD基类中定义如下：

```scala
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      SparkEnv.get.cacheManager.getOrCompute(this, split, context, storageLevel)//判断当前分区是否存在
    } else {
      computeOrReadCheckpoint(split, context)//如果有checkpoint就要从checkpoint恢复，否则就直接计算
    }
  }
```

然后调用map方法对每一个元素进行操作之后，返回一个新的iter对象。函数f定义了map中的具体操作。这里的map其实是Scala中Iterator类内部方法，其定义如下：
```scala
def map[B](f: A => B): Iterator[B] = new AbstractIterator[B] {
    def hasNext = self.hasNext
    def next() = f(self.next())
}
```