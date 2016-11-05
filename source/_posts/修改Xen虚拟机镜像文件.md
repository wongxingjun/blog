title: "修改Xen虚拟机镜像文件"
date: 2015-04-08 16:04:33
category: Xen
tags: [Xen,虚拟化]
---

上篇讲到Xen半虚拟化镜像的制作过程，本篇要介绍一下两个小技巧。有时候我们需要在没有开启虚拟机的情况下，对镜像里的文件进行修改。或者由于有新的需要，需要对镜像进行扩容。
<!--more-->

# 静态修改镜像文件
将镜像挂载到一个目录下，这里以/mnt为例
```
mount -o loop vm1.img /mnt
cd /mnt/
```
在/mnt下可以看到vm1的所有文件，可以进行操作，比如配置IP，环境变量等。不过完成之后一定要卸载一下，`umount vm1.img`

# 镜像扩容
## 追加镜像空间
```
dd if=/dev/zero bs=1024k count=4096 >> vm1.img
```
这里追加4096MB到vm1.img镜像中

## 扫描检查镜像
```
/sbin/e2fsck -f  vm1.img
```

## 调整文件系统大小
```
/sbin/resize2fs vm1.img
```
然后，你就会发现你的镜像长大了~~~