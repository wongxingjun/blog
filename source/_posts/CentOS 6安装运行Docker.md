title: "CentOS 6安装运行Docker"
date: 2016-03-16 15:32:18
categories: Docker
tags: [Docker,虚拟化]
---

最近需要用到Docker，奈何服务器上全是安装的CentOS6系统，Docker官方推荐的CentOS版本是7，但是很多旧机器依旧是5或者6系列，本文介绍如何在不进行系统升级的情况下在CentOS6中安装并运行Docker
<!--more-->

# 升级内核
按照官方给出的要求，内核至少要升级到3.10以上，这里使用的是3.10.58内核，解压内核源码
```shell
tar -xf linux-3.10.58.tar.xz
cd linux-3.10.58
cp /boot/config-2.6.32-431.el6.x86_64 ./
mv config-2.6.32-431.el6.x86_64 .config
sh -c 'yes "" | make oldconfig'
```
按照当前内核默认的配置，但是一定要注意，要加入Cgroups的支持
```
make menuconfig
```
General setup  --->
    Control Group support  --->
为方便起见，全选，然后保存，开始编译安装
```shell
make -j16 bzImage && make -j16 modules
make modules_install && make install 
```
安装完之后，添加grub启动引导项，略

# 安装Docker
重启进入新内核后开始安装Docker，按照官方给出的在CentOS下安装说明
```shell
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```
这里有个坑说明一下，就是在RedHat下安装，这里的*$releasever*要改成对应的版本号，比如6，不然就会定向到错误的源，然后安装Docker
```shell
yum install docker-engine
service docker start
```
然后进行运行测试，安装中遇到一个问题，这里记录一下。docker service启动之后，不能使用，说什么*docker dead but pid file exists
*，没办法就只能看日志了
![dockerlog](/figures/install-docker-on-centos/dockerlog.png)
结果发现应该是和libmapper有关，这是CentOS下docker运行的备库，所以安装升级一下
```shell
yum install device-mapper-event-libs device-mapper device-mapper-event device-mapper-libs
```
然后重启docker即可

# 小结
小记一笔流水账，后续将会介绍Docker的原理。