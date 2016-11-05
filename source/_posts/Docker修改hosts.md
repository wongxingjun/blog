title: Docker修改hosts
date: 2016-04-06 15:51:10
categories: Docker
tags: [Docker,虚拟化]
---

Docker修改hosts?这还不简单，打开vim直接敲就完事儿了！No,no,no,不是那么simple的。在很多场景中，比如我们需要搭建一个集群，这时候容器要识别集群内的节点，就需要添加相应的host解析。这时就需要修改容器的hosts文件，下面我们将会看到在Docker中自动化实现修改hosts不是那么简单的事。
<!--more-->

# 问题的由来
hosts文件其实并不是存储在Docker镜像中的，/etc/hosts, /etc/resolv.conf和/etc/hostname，是存在/var/lib/docker/containers/(docker_id)目录下，容器启动时是通过mount将这些文件挂载到容器内部的。因此如果在容器中修改这些文件，修改部分不会存在于容器的top layer，而是直接写入这3个文件中。容器重启后修改内容不存在的原因是每次Docker在启动容器的时候，Docker每次创建新容器时，会根据当前docker0下的所有节点的IP信息重新建立hosts文件。也就是说，你的修改会被Docker给自动覆盖掉。

# 解决办法
修改hosts一眼看上去是一件很容易的事，根据上面的分析其实不是那么简单的，如果一个分布式系统在数十个节点上，每次重新启动都要去修改hosts显得很麻烦，如何解决这一问题，目前有以下办法。

## 开启时加参数
开启容器时候添加参数--add-host machine:ip可以实现hosts修改，在容器中可以识别machine主机。缺点是很多个节点的话命令会很长，有点不舒服（当然，你可以脚本实现）

## 进入容器修改
进入容器后可以用各种编辑器编辑修改hosts文件，这样是最简单粗暴的方式，缺点就是每次都要修改一次。

## dnsmasq
在每个容器中开启dnsmasq开负责局域网内的容器节点主机名解析，但是这种方式在跨物理机的时候，好像不是很方便（具体情况还没有实践过，挖坑日后再填）。可以参考一下[kiwenlau的工作](https://github.com/kiwenlau/hadoop-cluster-docker)

## 修改容器hosts查找目录
我们可以想办法，让容器开启时候，不去找/etc/hosts文件，而是去找我们自己定义的hosts文件，下面是一个Dockerfile实例
```
FROM ubuntu:14.04
RUN cp /etc/hosts /tmp/hosts #路径长度最好保持一致
RUN mkdir -p -- /lib-override && cp /lib/x86_64-linux-gnu/libnss_files.so.2 /lib-override
RUN sed -i 's:/etc/hosts:/tmp/hosts:g' /lib-override/libnss_files.so.2
ENV LD_LIBRARY_PATH /lib-override
RUN echo "192.168.0.1 node1" >> /tmp/hosts #可以随意修改/tmp/hosts了
...
```
这样修改后，容器每次启动时就不去找/etc/hosts了，而是我们自己定义的/tmp/hosts，可以随心所欲在其中修改了

# 小结
以上问题可以用于跨主机部署分布式Docker集群，结合之前的跨物理机Docker网络互联，便可以实现快速高效部署分布式集群