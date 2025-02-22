---
layout: post
title: "Genymotion下载慢或者下载失败的解决办法"
date:  2018-01-08 22:50:07 +0800
categories: ["技术", "编程", "Linux和虚拟机"]
tag: ["Genymotion", "模拟器"]
---

转。原文地址：[https://blog.csdn.net/sean_css/article/details/52674091](https://blog.csdn.net/sean_css/article/details/52674091)

**办法如下：**

1. 首先点击界面上的 + 号（Add）按钮，选择你要下载的模拟器虚拟机版本，点击下载（一定要走这一步，不然会影响下面的步骤）

2. 然后到 `C:\Users\你的用户名\AppData\Local\Genymobile` 下面打开 `genymotion.log` 文件，找到 `Downloading file` 后面的 http 开头的链接，复制到迅雷开始下载；

3. 下载完成之后拷贝这个文件（称为文件1）到 `C:\Users\用户名\AppData\Local\Genymobile\Genymotion\ova` 。然后你会看到里面已经存在一个 `.ova` 文件，未下载完整的，用迅雷下载的替换掉里面的文件就行。

4. 重新打开 genymotion 模拟器，和刚才步骤1一样的步骤，你会发现很快就可以加载好了，接下来就可以愉快的使用模拟器了

**windows解决方案**
更新：
如果说你已经下载好了一个版本的模拟器 .ova 文件，那么如果你可以对其进行备份，下次直接拷贝进 `C:\Users\你的用户名 \AppData\Local\Genymobile\Genymotion\ova` 目录下面就可以了。这样就省去了再次下载的时间。