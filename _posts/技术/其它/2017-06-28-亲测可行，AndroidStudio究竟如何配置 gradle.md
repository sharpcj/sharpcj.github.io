---
layout: post
title: "亲测可行，AndroidStudio究竟如何配置 gradle"
date:  2017-06-28 18:36:07 +0800
categories: ["技术", "其它"]
tag: ["Hexo", "Android Studio"]
---

# 一、你不想看到的 Gradle Build Running
话说在天朝当程序员也是很不容易的，不管是查阅资料还是下载东西，很多时候你会发现自己上网姿势不对，当然对大多数程序员来说，这都不是事儿。这次重新安装了最新版的AndrodiStudio,按照国际惯例，第一次启动当然是按默认程序走一波 Hello World。可是，很有可能，你会看到你不想看到的如下界面：

![](/assets/images/技术/其它/亲测可行，Android%20Studio究竟如何配置gradle/pic1.jpg)

原因估计大家应该都知道，是你项目对应版本的 gradle 下载不下来造成的。在不改变上网环境的情况下，解决办法就是下载 gradle 到本地，然后做相应配置。下面主要说说怎么配置。

# 二、亲测可行的解决方案
## 2.1解决问题
打开项目中的 gradle-wrapper.properties 文件，如下：

![](/assets/images/技术/其它/亲测可行，Android%20Studio究竟如何配置gradle/pic2.jpg)

意思就是在 `GRADLE_USER_HOME/wrapper/dists/` 下面去找对应的 gradle 文件，没有的话，就去
到最后一行
`distributionUrl=https\://services.gradle.org/distributions/gradle-3.4.1-all.zip`
中的地址下载，其中 gradle-3.4.1-all.zip 这个说明你当前工程配置的 gradle 的版本为 3.4.1。所以需要下载该版本的gradle，
你可以到这里下载：
[http://services.gradle.org/distributions/](https://services.gradle.org/distributions/)

如果访问速度不佳，国内有镜像地址：
[腾讯云镜像 Gradle下载地址: mirrors.cloud.tencent.com/gradle/](https://mirrors.cloud.tencent.com/gradle/)

接下来打开 AndroidStudio 中 gradle 的设置界面，如下：

![](/assets/images/技术/其它/亲测可行，Android%20Studio究竟如何配置gradle/pic3.jpg)

可以看到，默认的 gradle 的目录是 `C:/Users/SharpCJ/.gradle`，进入该目录
`C:\Users\SharpCJ\.gradle\wrapper\dists\gradle-x.x-all\`，可以看到有一串看起来像乱码字符的文件夹，进入，删掉里面的 gradle-x.x-all.zip.lck 和 gradle-x.x-all.zip.part 文件，然后把前面下载下来的对应的 gradle-x.x-all.zip 文件放进去，不用解压,然后 Ctrl+F9, 重新编译工程，则会自动解压。OK,问题解决了。

## 2.2 更改 gradle 版本
假设现在要自己改变 gradle 版本，同样的道理，改 gradle-wrapper.properties 文件中最后一行版本号，然后编译则会生成对应的乱码字符的文件夹，然后按上面的操作进行，**注意不能手动新建文件夹**。
但是有时候，你会发现，编译的时候仍然会报错，这时候，很有可能是你选择的 gradle 版本太低了。gradle的版本还需要和 gradle 插件的版本对应，提高 gradle 版本即可。

# 三、 gradle 和 gradle 插件的区别
我们知道，AndroidStudio 是基于 gradle 构建项目的，安装 gradle插件 才能使系统能支持运行 gradle。安装 AndroidStudio 后就已经帮我安装了 gradle插件．但 gradle插件是独立于Android Studio运行的,所以它的更新也是与 AndroidStudio 分开的。
打开工程的 build.gradle 文件，能看到如下界面：

![](/assets/images/技术/其它/亲测可行，Android%20Studio究竟如何配置gradle/pic4.jpg)

这个就是 gradle插件的版本号。下图展示了 gradle插件 和 gradle 之间的对应关系：

| plugin version | Required Gradle version |
| ----------- | ----------- |
| 1.0.0-1.1.3 | 2.2.1-2.3 |
| 1.2.0-1.3.1 | 2.2.1-2.9 |
| 1.5.0 | 2.2.1-2.13 |
| 2.0.0-2.1.2 | 2.10-2.13 |
| 2.1.3+ | 2.14.1+ |

因为 gradle 在不断更新，自然 gradle插件也需要不断更新版本才能提供对新版本gradle的支持，所以最好让你的Gradle和Gradle插件都更新到最新。
更新 gradle 插件的方法：
通过选择 File > Project Structure > Project 来指定Gradle版本，然后点击 Tools > Android > Sync Project with Gradle Files 去下载。