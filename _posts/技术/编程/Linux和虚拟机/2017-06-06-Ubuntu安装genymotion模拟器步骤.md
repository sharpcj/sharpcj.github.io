---
layout: post
title: "Ubuntu安装genymotion模拟器步骤"
date:  2017-06-06 15:36:07 +0800
categories: ["技术", "编程", "Linux和虚拟机"]
tag: ["genymotion", "模拟器", "虚拟机"]
---

Ubuntu安装genymotion模拟器步骤:
1. 安装VitrualBox
genymotion模拟器需要有VirtualBox环境，打开终端（ctrl + alt + T），执行以下命令：

```
sudo apt-get install virtualbox
```

2. 下载genymotion模拟器相应版本
cd 进入安装包路径，依次执行：

```
chmod +x genymotion-2.2.2_x64.bin（赋予执行权限）

./genymotion-2.2.2_x64.bin（进行安装）
```

解压之后，进入解压后的目录，点击genymotion，打开，剩下的就是图形化操作了。

如果遇到模拟器不能start，解决办法：

```
-apt-repository ppa:ubuntu-toolchain-r/test

sudo apt-get update

sudo apt-get install gcc-4.9 g++-4.9

sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.9 60 --slave /usr/bin/g++ g++ /usr/bin/g++-4.9
```

原因是Ubuntu 默认使用GCC 4.8.4 而缺少下面列出来的文件:

```
/usr/lib/libstdc++.so.6: version `GLIBCXX_3.4.20' not found
```

如果Ubuntu是虚拟机。开启 CPU 支持。