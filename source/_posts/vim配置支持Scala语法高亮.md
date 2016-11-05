title: "vim配置支持Scala语法高亮"
date: 2015-04-15 17:38:55
category: 配置
tags: [工具, Scala, 插件]
---

在老大的感染下，我成了一个vim党员，其实有很多技巧还不是很熟。因为在看Spark相关的一些东西有时候需要在服务器上看Scala代码，vim默认不支持Scala高亮，这里给出配置方法，作为笔记吧。
<!--more-->

# 安装Vundle
插件神器[Vundle](https://github.com/gmarik/Vundle.vim)，你值得拥有。

# 下载scala.vim
安装直接用shell脚本
```shell
mkdir -p ~/.vim/{ftdetect,indent,syntax}
for d in ftdetect indent syntax
    do wget --no-check-certificate -O ~/.vim/$d/scala.vim \
       https://raw.githubusercontent.com/derekwyatt/vim-scala/master/$d/scala.vim;
    done
```

# 安装vim-scala插件
安装完成后，在.vimrc配置文件中添加：
```
Plugin 'derekwyatt/vim-scala'
```
在vim下执行安装：
```
:BundleInstall
```

大功告成，上图
![vim-scala](/figures/vim/vim-scala.jpg)
