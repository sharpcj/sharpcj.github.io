---
layout: post
title: "virtualbox安装增强功能时【未能加载虚拟光盘】"
date:  2018-04-11 09:50:07 +0800
categories: ["技术", "编程", "Linux和虚拟机"]
tag: ["virtualbox", "虚拟机"]
---

本文转载自链接: [https://zycao.com/virtualbox-ubuntu-vboxsguestadditions.html](https://zycao.com/virtualbox-ubuntu-vboxsguestadditions.html)

今天在使用Virtualbox中的Ubuntu虚拟机，打算作为微丫头本地测试，结果屏幕分辨率比较低，不方便使用，就想安装增强功能来实现更改分辨率，
但是在安装时出错：未能加载虚拟光驱 VBoxsGuestAdditions.iso到虚拟电脑

![](/assets/images/技术/编程/Linux和虚拟机/virtualbox安装增强功能/pic1.jpg)

经过折腾，最后通过互联网找到了解决方法：
进入系统在侧边找到如图加载的虚拟光驱，右击，点击弹出，然后就可正常安装增强功能了

![](/assets/images/技术/编程/Linux和虚拟机/virtualbox安装增强功能/pic2.jpg)

点击安装增强功能

![](/assets/images/技术/编程/Linux和虚拟机/virtualbox安装增强功能/pic3.jpg)

点击“运行”

![](/assets/images/技术/编程/Linux和虚拟机/virtualbox安装增强功能/pic4.jpg)

输入登录系统的密码，点击授权，就开始自动安装了

![](/assets/images/技术/编程/Linux和虚拟机/virtualbox安装增强功能/pic5.jpg)

如图，为安装界面，安装完成后按下回车键，就按照成功了。

![](/assets/images/技术/编程/Linux和虚拟机/virtualbox安装增强功能/pic6.jpg)

安装好后，虚拟机可以在无缝模式和自动显示尺寸下运行了。