title: "Xen半虚拟化镜像制作"
date: 2015-04-08 15:50:21
category: Xen
tags: [Xen,虚拟化]
---

本文主要介绍Xen下半虚拟化Guest OS镜像的制作过程。
<!--more-->

# 创建映像文件
```
dd if=/dev/zero of=vm1.img bs=1M seek=8192 count=1
```
创建大小为 8G ，名为 vm1.img 的映像文件

# 格式化映像为linux文件系统
```shell
/sbin/mkfs.ext3 vm1.img
```
提示 Proceed anyway? (y,n) 输入y 回车就可以了

# 挂载映像
```
mkdir /mnt/vm1
mount -o loop vm1.img /mnt/vm1
```
这样我们就可以向 vm1.img中存放文件了

# 拷贝系统文件到虚拟磁盘中
将物理机里面的文件拷贝到 /mnt/vm1中。如下：
```
cp -ax /{root,dev,var,etc,usr,bin,sbin,lib,lib64} /mnt/vm1/
mkdir /mnt/vm1/{proc,sys,home,tmp}
```

# 修改/mnt/vm1/etc/fstab文件
```
echo "/dev/xvda1 / ext3 defaults 1 1" > /mnt/vm1/etc/fstab
```
xen4.0不支持hda ，sda，要改成 xvda
否则会出现如下错误：
```
mount : could not find filesystem '/dev/root' 
setup other filesystem 
setting up now root fs 
set up root :moving /dev faild:No such file or directory 
no fstab.sys,mounting inernal defaults 
setuproot:error mounting /proc :No such file or directory 
setuproot:error mounting /sys:No such file or directory 
switching to new root and running init 
umounting old /dev
umounting old /proc 
umounting old /sys
switchroot : mount faild : No such file or directory 
kernel panic:not syncing :attempted to kill init
```

# 卸载/mnt/vm1：
```
umount /mnt/vm1
```
到此半虚拟的镜像就制作完成

# 修改配置文件
```
cp /etc/xen/xmexample1 ./vm1.cfg
vim vm1.cfg
```
修改完成后内容如下 ,括号里面为注释：
```
kernel = "/boot/vmlinuz-2.6.31.13" （虚拟机内核） 
ramdisk = "/boot/initrd-2.6.31.13.img" （虚拟机的内存虚拟磁盘） 
memory = 512（指定虚拟机的内存大小为 512M）
name = ”pvops“ （虚拟机的名字） 
vcpus = 2 （指定虚拟机的 cpu个数为2 个）
#vif = [ 'mac=00:16:3e:00:00:99, bridge=xenbr0' ] （网卡参数） 
disk = [ 'file:/root/vm1.img,xvda1,w' ] （虚拟机磁盘，将文件vm1.img映射成 xvda1)
root = "/dev/xvda1 ro" (虚拟机从xvda1 启动)
```
这里的 root="/dev/xvda1 ro" 要和第5步中修改的 fstab里面写的一模一样，否则就无法启动

Extra="4 console=hvc0"
# 启动虚拟机：
```
xm create vm1.cfg
```






--------------------------Update--------------------------
# 开启虚拟机如果遇到错误
```
[   75.915187] dracut Warning: Initial SELinux policy load failed.
[   75.915223] dracut: FATAL: Initial SELinux policy load failed. Machine in enforcing mode. To disable selinux, add selinux=0 to the kernel command line.
[   75.915234] dracut: Refusing to continue
[   75.915376] dracut Warning: Signal caught!
[   75.917802] dracut Warning: dracut: FATAL: Initial SELinux policy load failed. Machine in enforcing mode. To disable selinux, add selinux=0 to the kernel command line.
[   75.917838] dracut Warning: dracut: Refusing to continue
[   75.918033] Kernel panic - not syncing: Attempted to kill init!
[   75.918041] Pid: 1, comm: init Not tainted 2.6.31.8 #
```
在虚拟机配置文件后加一行禁用selinux
```
extra="selinux=0"
```

# 用和制作镜像的内核不一样的内核版本启动虚拟机
这时候需要把物理机上对应的内核模块拷贝进虚拟机镜像
```
cp -ax /lib/modules/(内核版本)  /mnt/vm1/lib/modules/
```