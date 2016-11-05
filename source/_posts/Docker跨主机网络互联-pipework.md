title: Docker跨主机网络互联---pipework
date: 2016-04-06 09:05:04
categories: Docker
tags: [Docker,虚拟化]
---

Docker最新版1.10中已经可以实现自主跨主机网络互联了，但服务器上是CentOS6下安装的，版本最高是1.7.1，之前用weave实现了网络互联，但需要开启两个容器，这里记录pipework来实现跨主机联网。
<!--more-->

# 准备工作
主机node216(IP=192.168.226.216),主机node214(IP=192.168.226.214)，均安装配置好Docker

# 安装pipework
安装过程较为简单，分别执行
```shell
git clone https://github.com/jpetazzo/pipework.git
cp -rp pipework/pipework /usr/local/bin/
```

# 配置网桥
pipework不适用默认的docker0网桥，自己配置一个，以下是一个例子，ifcfg-br0
```
DEVICE="br0"    
TYPE="Bridge"
ONBOOT="yes"
IPADDR=192.168.226.214
NETMASK=255.255.255.0
DNS1=8.8.8.8
```
将之桥接到网卡eth0，将之前的IP注释掉
```
NAME="enp0s25"
DEVICE="enp0s25"
ONBOOT=yes
IPV6INIT=no
BOOTPROTO=static
#IPADDR=192.168.226.214
NETMASK=255.255.255.0
TYPE=Ethernet
BRIDGE=br0 #一定要有这个！
```

# 开启容器
开启容器并配置IP的过程很简单，在node214上
```shell
pipework br0 $(docker run -itd --name test1 ubuntu:14.04 bash) 192.168.0.1/24
docker exec -it test1 bash
```
同样，在node216上
```shell
pipework br0 $(docker run -itd --name test2 ubuntu:14.04 bash) 192.168.0.2/24
docker exec -it test2 bash
```
在node214上的容器内测试
```
root@b5cde5e57712:/# ping 192.168.0.2
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.263 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.260 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=0.274 ms
64 bytes from 192.168.0.2: icmp_seq=4 ttl=64 time=0.260 ms
64 bytes from 192.168.0.2: icmp_seq=5 ttl=64 time=0.252 ms
```
可以看到，两个主机上的容器实现了互联

# 注意
pipework开启容器时候，有可能遇到错误，提示
```
Object "netns" is unknown, try "ip help".
```
这是因为缺少支持net namespace造成，有两种办法，升级内核或者iproute，后者较为简单，查看发现本机上安装的是iproute-2.6.32-45.el6.x86_64，安装较为简单
```
wget https://repos.fedorapeople.org/openstack/EOL/openstack-grizzly/epel-6/iproute-2.6.32-130.el6ost.netns.2.x86_64.rpm
rpm -ivh ./iproute-2.6.32-130.el6ost.netns.2.x86_64.rpm
```
然后就不会有提示错误出现了

# 小结
和weave相比，这种方式实现跨主机网络互联较为简单，推荐pipework。