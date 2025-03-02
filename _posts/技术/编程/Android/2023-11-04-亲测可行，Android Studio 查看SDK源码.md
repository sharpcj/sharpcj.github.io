---
layout: post
title: "亲测可行，Android Studio 查看源码出现 Source for ‘Android API xxx Platform’ not found 的解决方法"
date:  2023-11-04 18:58:12 +0800
categories: ["技术", "编程", "Android"]
tag: ["Android"]
---

亲测可行，Android Studio 查看源码出现 Source for ‘Android API xxx Platform’ not found 的解决方法

如标题中的问题，产生的原因就是 SDK 源码目录下找不到对应版本的源码文件。解决方案一般就是下载对应版本的源码文件即可。

这里主要是另一种情况，每次 Google 发布 Android 新的版本时，对应源码还没有提供下载（一般会在正式版发布以后的某个时段提供）。这时怎么办呢？

思路就是把旧版本的源码先用着。

这里以 Android API 34 为例。，将 Android 33 的源码强行拷贝，当做 API 34 来用。

步骤如下：

1. 到 Android SDK 目录下(sdk/sources) 下复制 android-33 并修改为 android-34.

2. 修改 android-34 中的 `package` 和 `source.properties` 文件，将其中所有的 33 改为 34

![](/assets/images/技术/编程/Android/查看%20SDK%20源码/pic1.png)

3. 修改 `jdk.table.xml` 文件,把所有 Android API 34 Platform 的 标签的路径改为 android-34 的路径。该文件路径为：

```
C:/Users/.AndroidStudio{version}/config/options/jdk.table.xml
```
![](/assets/images/技术/编程/Android/查看%20SDK%20源码/pic2.png)

4. 重启 Android Studio，便可以看到源码了。