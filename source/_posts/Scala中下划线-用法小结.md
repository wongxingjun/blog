title: Scala中下划线_用法小结
date: 2016-07-07 18:08:10
categories: 编程
tags: [编程, Scala]
---

Scala函数式编程确实可以有效提高编程效率，和其祖先Java类似，Scala也有一堆通配符，而且更复杂、高级，本文对目前自己常见的下划线_的用法进行一个小结，备查。<!--more-->

# 导入包通配符
```scala
import scala.collection.mutable._
```
# 隐藏导入
```scala
import org.th.Fun.{Bar => Foo, _}
```

# 可忽略参数
```scala
val a = Array(1,2,3)
a.map(_ => println("Hello"))
```

# 高阶类型参数
```scala
case class A[K[_], T](a: K[T])
```

# 临时变量
```scala
val _ = 10
```

# 通配模式
```scala
a match{
	case Some(i) => i
	case _ =>
}
```

# 占位符
```scala
val a = List(1,2,3)
a.map(_ + 2)
```

# 默认初始化
```scala
val a = _
```

# 部分函数与偏函数
```scala
def func(a:Int, b:Int){
	a+b
}
val func2 = func(2, _:Int) //部分函数

List(1, 2, 3) foreach println _
```

# 参数名
```scala
val a = Map((1,"20160706"),(2,"20150902"))
for( i <- a)
	println(e._2)
```

# 参数序列
```scala
List(1 to 100:_*)
```

**To be continued**