title: "vim安装YouCompleteMe"
date: 2015-04-20 20:11:51
category: 配置
tags: [工具,插件]
---

代码补全对vim党来讲是最重要的，可惜vim自带的代码补全实在很不堪。目前而言，[YouCompleteMe](https://github.com/Valloric/YouCompleteMe)插件是最为牛X的。由于要在服务器上写C和Python等，自动补全不能没有。这个插件在Ubuntu下安装十分简单，但是在CentOS下，就有那么一点点麻烦了，这里记录一下。
<!--more-->

# 准备工作
使用Vundle来管理插件，安装Vundle这个不再赘述。此外需要vim7.3.584+的vim并支持python。还要安装Clang3.2+。如果编译升级vim，那么一定要./configure `--enable-pythoninterp`这个选项，然后再make，make install。

# 安装YouCompleteMe
下载源码：
```
git clone https://github.com/gmarik/vundle.git ~/.vim/bundle/vundle  
```
进入`.vim/bundle/YouCompleteMe`下，开始安装
```
./install --clang-completer --system-libclang
```
提示让你干嘛就干嘛，安装完成后，在.vimrc下添加：
```
Bundle 'Valloric/YouCompleteMe'
```
使插件生效。然后添加一点配置（这个配置是很简单的功能，复杂的功能还在探索中）
```
let g:ycm_global_ycm_extra_conf = '~/.vim/bundle/YouCompleteMe/third_party/ycmd/cpp/ycm/.ycm_extra_conf.py'
let g:ycm_collect_identifiers_from_tags_files = 1
let g:ycm_seed_identifiers_with_syntax = 1
let g:ycm_confirm_extra_conf = 0
```
在~/.vim/bundle/YouCompleteMe/third_party/ycmd/cpp/ycm/.ycm_extra_conf.py中添加
```
'-isystem'
'/usr/include'
```
这样才能支持C++标准库补全

完成之后就可以使用一般的自动补全功能了，上图
![vim-ycm](/figures/vim/vim-ycm.png)
