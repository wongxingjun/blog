title: "Paxos算法的一种简单理解"
date: 2015-05-18 19:46:14
categories: 分布式
tags: [分布式,算法]
---

Lamport提出的[Paxos](http://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)算法是一种基于消息传递且具有高度容错特性的一致性算法,被誉为分布式系统的基石。目前已经有很多成熟的分布式系统使用Paxos，如Google的Chubby、Megestore和Spanner，Zookeeper等。因为Lamport在[论文](http://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/past/03F/notes/paxos-simple.pdf)中解释这个算法时将其描述成一种游戏，一般人从论文中很难将其和分布式系统中的一致性联系起来。搞不清楚Paxos算法的目的和在分布式存储系统中的应用（Lamport觉得是人们无法理解他的幽默）。<!--more-->
最近学习分布式一些基础知识接触到这个有趣的算法，和大多数人一样看完Lamport的论文也是一头雾水，因此就尝试自己去理解Paxos。从问题出发，以实现一个简单的分布式存储系统为切入点，简单梳理一下Paxos的基本原理。

# 一致性问题
一致性问题简言之，就是要确定一个不可变变量的值，然后保证不会被随意更改，这样可以保证分布式系统中多个机器上进行的操作是一致的，不会因为各种因素影响，导致大家得到的取值不一致，最后出现乱套的情况。对这个问题我们做以下定义：
- 不可变变量的取值可以是任意二进制数。
- 一旦取值确定，将不再更改，并且可以被获取到（不可变性，可读性）。

# 在分布式存储系统中的应用
- 存储系统中数据本身可变，目前绝大多数分布式存储系统为了保证数据的高可用，通常会将数据保留多个副本，这些副本会放置在不同的物理的机器上。
- 对多副本的更新操作序列(OP0,OP1,OP2...)，如果不进行任何控制，那么网络延迟、超时、节点宕机等因素会导致各个副本之间的更新操作不一样。在每一步多个副本的操作不一样，我们希望每个副本的操作是一样的，保证多个副本的一致性。
- 用Paxos依次来确定不可变变量OPi的取值（即第i个操作是什么）。
- 每确定完OPi之后，让各个副本依次执行OPi，依次类推。

# 系统的定义
确定一个不可变变量取值问题的定义：设计一个系统来存储一个var变量，系统应该满足
- 系统内部由多个acceptor组成
- 系统外部由多个proposer任意并发调用系统API，向系统提出修改var值
- var可以是任意二进制数据
- 系统对外的API接口为propose(var,V) => [ok,f] or [error]

为了保证数据一致性，这里var的取值应该满足：
- var取值没有确定时候，置为null
- var一旦被确定，就不能再被别的proposer更改，并且可以被获取

此外还应该保证系统容错性，即：
- 系统应该容忍任意proposer机器发生故障不能工作
- 系统能容忍少数（即半数以下）acceptor发生故障

# 系统解决方案
## 方案一：简单互斥访问锁机制
考虑系统中有单个acceptor，可以采用最简单的互斥访问方式来实现一致性，即通过锁机制来管理并发的proposer。
acceptor保存的状态和提供的接口：
- acceptor保存var和一个互斥锁lock，[var, lock]
- acceptor::prepare() => 加互斥锁，给var互斥访问权，并返回var当前取值f。如果已经加锁了，就返回error
```cpp
    if lock==true
        return error
    else
        lock=true
        access=true
        return [true, f]
```

- acceptor::release() => 解互斥锁，收回var的互斥访问权，返回[ok, f]
```cpp
    lock=false
    access=false
    return [ok, f]
```

- acceptor::accept(var,V) => 如果已经加锁，且var没有确定取值，就接受更新var为V
```cpp
    if lock==true&&var==null
        var=V
    else
        return error
```

propose(var,V)分两阶段运行：
- proposer通过acceptor::prepare()向accptor申请访问权和var取值，如果申请失败返回error
- proposer一旦获得权限，就检查var值是否被确定，如果是null，就调用acceptor::accept(var,V)提交修改成自己提交的V。如果var不为空就说明之前有proposer已经提交了var，当前proposer就必须通过acceptor::release()释放访问权。
```cpp
    [access, var]=acceptor::prepare()
    if access==true
        if var==null
            acceptor::accept(var, V)
        else
            acceptor::release()
    else
        return error
```

**这种方案存在一个问题，就是proposer获取互斥访问权之后，如果还没有释放之前就挂掉了，其他proposer将会一直等待访问权，发生死锁。也就是说在这种方案下不能容忍proposer挂掉**

## 方案二：抢占式访问机制
基于方案一存在的问题，引入抢占式访问权，acceptor可以让proposer互斥访问权失效，然后可以将互斥访问权发放给其他proposer。其具体实现是：
- proposer向acceptor申请访问权时指定一个标记，称之为epoch值，获取到访问权之后才能提交修改var
- acceptor采用“喜新厌旧”原则，一旦收到更新的访问权申请（也就是更大的epoch值，简单来讲，可以用系统时间来作为epoch），就立即让旧的访问权失效，之前获取访问权的proposer再无法访问。然后向新的epoch发放访问权限，只接受新的epoch提交修改var值

**新的epoch抢占旧的epoch，让旧的epoch访问权失效，proposer无法运行**
为了保证一致性，不同的epoch和proposer之间采用“后者认同前者”的原则：
- 在确定旧的epoch无法为var赋确定性取值时，新的epoch才会提交修改var值
- 一旦旧的epoch为var赋了确定性取值，新的epoch是不能提交修改的，并且能够获取到这个var值

acceptor保存的状态和提供的接口：
- 当前var取值和接受的epoch，[accepted_epoch, accepted_var]
- 最新发放访问权的epoch，latest_prepared_epoch
- acceptor::prepare(epoch)
```cpp
    if latest_prepared_epoch < epoch
        给予访问权给epoch, access(epoch)=true
        更新latest_prepared_epoch=epoch
        return [true, var]
    else return error
```

- acceptor::accept(var,prepared_epoch,V)
```cpp
    if latest_prepared_epoch == prepared_epoch
        更新var，[accepted_epoch, accepted_var] = [prepared_epoch, V]
    else return error
```

propose(var, V)的运行：
- 获取epoch轮次的访问权和当前var值
```cpp
    if acceptor::prepare(epoch)==ture
        return [true, accepted_var]
    else return error
```

- 如果确定旧的epoch无法生成var取值，当前epoch就提交更新var，否则就认同已经存在的var
```cpp
    if var==null:
        return acceptor::accept(var, epoch, V)
    else return [ok, accepted_var]
```

**这个方案也存在一个问题，那就是只有一个acceptor，一旦挂掉，整个系统就会宕机**

## 方案三：Paxos
方案二中单个acceptor挂掉会导致系统宕机，Paxos就是在此基础上引入多acceptor。这样就会带来一个问题，那就是var取值到底什么时候是确定、不可更改？Paxos中采用少数服从多数的原则，**如果半数以上的acceptor接受了一个epoch提交的var取值f，那么就视f为确定性取值，不能被更改**

propose(var, V)的运行：
- 选定epoch，获取半数以上acceptor访问权和对应的一组var取值
```cpp
    if acceptor::prepare(epoch)==[true,num] && num >= 1/2*numOfAcceptors
        return [List(acceptor), List(var)]
    else return error
```

- 如果获取的List(var)都为空，说明之前的epoch无法为var确定取值，此时当前epoch要努力使得[epoch, f]被系统接受，对var进行赋值。如果List(var)不为空，那就判断是否过半数，过半数就放弃赋值，否则提交[epoch, f]
```cpp
    if List(var)==null
        return acceptor::accept(epoch, f)
    else if sizeof(List(var)) >= 1/2*numOfAcceptors
        return [ok, accept_var]
    else
        return acceptor::accept(epoch, f)
```

# 小结
以上是单轮达成一致性协议的Paxos实现过程，对于多轮达成一致性，也可以类比理解。至此对Paxos终于有了一个相对清晰的了解，Stanford的Diego Ongaro和John Ousterhout提出的新的一致性算法[Raft](https://raftconsensus.github.io/)，相对Paxos比较好理解一点（论文[In Search of an Understandable Consensus Algorithm](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf)），后期也会学习一下。当然，分布式系统不仅仅包括一致性问题，还有很多十分有意思的东西，需要我们去探索和学习。
