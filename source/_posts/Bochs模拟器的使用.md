title: "Bochs模拟器的使用"
date: 2015-06-18 15:09:48
categories: Linux
tags: [工具]
---

很早之前就在琢磨，能不能自己撸一个简单的操作系统内核出来，但是一直没有实质性行动。最近受师兄的启发，要开始造轮子了，不然编程能力是很难有提高的，就重拾当初的想法了。<!--more--> 做一个简单的操作系统内核出来，要保证能够正常运行，当然不能直接在物理机上测试（这是显然了，不然你怎么跟踪错误，一次次重启去看dump是一件很心累的事，而且也没额外的电脑），所以必须用到模拟器，采用Bochs或者Qemu都可以，这里给出Bochs的安装、基本使用测试，为后期的工作做好准备。

# 编译安装
我采用的环境是Ubuntu-14.04 X86_64，默认的apt-get安装的Bochs是不带有与gdb等工具联合调试功能的，所以必须自己动手来编译安装。去[sourceforge](http://sourceforge.net/projects/bochs/)下载源码，我使用的是2.6.8版本。需要提前安装g++，build-essential，xorg-dev，直接apt-get安装即可，然后
```shell
$sudo apt-get install build-essential
$sudo apt-get install xorg-dev
$tar -xvf bochs-2.6.8.tar
$cd bochs-2.6.8
$./configure --prefix=/opt/tools/bochs  --enable-gdb-stub  --enable-disasm
$sudo make && make install
```
安装完成，建立软链接
```shell
$sudo ln -s /opt/tools/bochs/bin/bochs /usr/bin/bochs
$sudo ln -s /opt/tools/bochs/bin/bximage /usr/bin/bximage
```
大功告成

# 制作虚拟软盘
执行bximage
![flappya](/figures/bochs-emulator/flappya.png)


# 写启动测试程序
写一个简单的开机启动执行的汇编代码，简单打印一句话
```asm
org     07c00h
        mov ax,cx
        mov ds,ax
        mov es,ax
        call DispStr
        jmp $

DispStr:
        mov ax,BootMessage
        mov bp,ax
        mov cx,14
        mov ax,01301H
        mov bx,000CH
        mov dl,0
        int 10H
        ret
BootMessage: db "Hello,Xingjun!"
times 510-($-$$) db 0
dw 0xAA55
```

# 编译并写入软盘
```
nasm boot.asm -o boot.bin
dd if=boot.bin of=a.img bs=512 count=1
```

# Bochs配置
编写bochs配置文件，这是用来启动bochs的，存为bochsrc
```
# how much memory the emulated machine will have
megs: 32

# filename of ROM images
romimage: file=/opt/tools/bochs/share/bochs/BIOS-bochs-latest
vgaromimage: file=/opt/tools/bochs/share/bochs/VGABIOS-lgpl-latest

# what disk images will be used
floppya: 1_44=a.img, status=inserted

# choose the boot disk.
boot: a

mouse: enabled=0
```

# 启动测试
执行bochs -f bochsrc，即可看到输出提示，直接回车（默认6，开始模拟）
![simulate](/figures/bochs-emulator/bochs.png)
