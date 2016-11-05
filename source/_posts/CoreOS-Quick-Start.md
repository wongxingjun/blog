title: CoreOS Quick Start
date: 2015-12-01 11:39:21
categories: Docker
tags: [Docker,虚拟化]
---

Docker作为最火的容器解决方案自2013年发布以来取得了极大的关注度，不管是工业界还是学术界对其热情高涨。其轻量、快速等特点让其一跃成为取代传统虚拟机的更为便捷的应用部署方式。CoreOS是一种专门为Docker量身打造的Linux系统，自带Docker运行环境，可以极大地方便Docker环境搭建，此文为尝鲜版。<!--more-->

# 环境
这里在Windows上以VirtualBox来运行CoreOS虚拟机来做尝试。因此需要以下工具：
[VirtualBox](https://www.virtualbox.org/wiki/Downloads)
[CoreOS VirtualBox Image](http://stable.release.core-os.net/amd64-usr/current/)

# 准备工作
VirtualBox的安装很简单，不做介绍。主要是CoreOS的镜像使用。下载镜像和虚拟机配置文件，文件名分别为coreos_production_virtualbox_image.vmdk.bz2和coreos_production_virtualbox.ovf，放在同一目录下，用VirtualBox直接导入虚拟机即可。
![import](/figures/coreos-quick-start/import-vm.png)

# 进入系统
准备工作完成后直接开机进入系统，直接进入后默认会要求填入用户名和密码登陆，无法知道用户名和密码（README没有提？）
![enter-os](/figures/coreos-quick-start/enter-os.png)
在grub中进入终端，修改用户和密码，编辑default登陆选项，在最后添加'console=tty1 coreos.autologin=tty1'，F10开机进入终端模式，sudo passwd core修改用户密码。
![login-tty1](/figures/coreos-quick-start/tty1-login.png)
![login-tty1](/figures/coreos-quick-start/tty1-login-OK.png)
完成修改之后重启，然后就可以顺利登陆系统了。

# Docker初体验
CoreOS默认自带有Docker整套运行环境，甚至连包管理器都没有，就是为了尽可能减少系统复杂度，体现出高度的专用性。直接命令行
```
docker run -it ubuntu
```
![ubuntu](/figures/coreos-quick-start/docker-run-ubuntu.png)
这样就可以畅玩Docker了，后续会补上怎样完整地将自己的应用部署到一个Docker容器中。
