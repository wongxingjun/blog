title: "Docker跨主机网络互联---weave"
date: 2016-03-23 11:44:36
categories: Docker
tags: [Docker,虚拟化]
---

Docker脱胎于LXC项目，充分利用Linux内核中的Cgroups机制，并进行一系列的封装，在保证性能的基础上极大增强了其易用性和便捷性，因此迅速风靡业界。但Docker也存在一些短板，比如跨物理机之间的网络互联，因为Docker默认是采用NAT方式访问外部网络，这样便造成了难以直接和别的物理机上的容器互联。跨主机互联的解决办法有多种，这里记录使用weave来解决这一问题的过程。
<!--more-->

# 准备工作
主机node215(IP=192.168.226.215)，node216(IP=192.168.226.216)，均安装配置好Docker。

# 安装weave
分别在两台主机node215和node216上执行
```
sudo wget -O /usr/local/bin/weave https://raw.githubusercontent.com/zettio/weave/master/weave
sudo chmod a+x /usr/local/bin/weave
```

# 开启weave
weave需要开启两个容器来辅助，因此会拉取两个镜像下来，首次执行weave launch便会拉取相关镜像，两个主机都要开启
```
[root@node216 mydockerbuild]# weave launch
Unable to find image 'weaveworks/weaveexec:latest' locally
latest: Pulling from weaveworks/weaveexec
2a250d324882: Pull complete 
a2fe59ddea2a: Pull complete 
b15851d0d54c: Pull complete 
d1e7f59bcd61: Pull complete 
3deea8a5dcdf: Pull complete 
```

# 开始使用
在主机node216上，我们需要手动和node215连接
```
weave connect 192.168.226.215
```
这样才能在容器中使用weave互联
在node216上开启一个容器，指定其IP为192.168.0.2
```
weave run 192.168.0.2/24 -it ubuntu bash
```
在node215上开启一个容器，指定其IP为192.168.0.3
```
weave run 192.168.0.3/24 -it ubuntu bash
```
然后在node215上的容器内测试
```
root@f8d5d1a09f9a:/# ping 192.168.0.2
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=1.35 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=1.10 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=1.16 ms
64 bytes from 192.168.0.2: icmp_seq=4 ttl=64 time=0.921 ms
64 bytes from 192.168.0.2: icmp_seq=5 ttl=64 time=1.23 ms
```
可以看到，实现了跨主机的网络互联

# 小结
Docker的网络和IO一直被人所诟病，在社区没有提出完美解决方案之前，还是得用这种麻烦的方式解决，另外还有pipework等方式，后续也会尝试体验一下。