---
layout: post
title: "LM Studio + open-webui 快速本地部署大语言模型"
date:  2024-07-22 17:58:12 +0800
categories: ["技术", "AIGC", "大模型"]
tag: ["AIGC", "LLM"]
image:
  path: /assets/images/技术/AIGC/大模型/LLM/LLM.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: LM Studio + open-webui 本地部署大模型.
---

## 一、前言
自 OpenAi 发布 ChatGPT 对话性大语言模型，AI 这两年发展迎来爆发，国内外也衍生了大量的语言模型开放给公众使用。为了可以让更多人接触到AI，让本地化部署更加轻便快捷，于是就有了Ollama、LM Studio等可以在本地部署模型的工具。

相比之下，ollama 需要通过命令进行安装，下载模型，以及对话， 如果需要 web 界面，可搭配 open-webui 进行配套使用，整套流程下来虽算不上复杂，但是对于没有编程经验的人来说，还是需要花费一些时间的。而 LM Studio 对小白用户更加友好方便，LM Studio 直截了当提供了图形化界面，并且直接下载 gguf 模型文件加载就可以直接使用了。当然也可以搭配 open-webui 进行网页版界面使用。

## 二、环境准备
系统：Windows\支持Apple M系列芯片\Linux系统

CPU：支持AUX2指令即可

内存：16G及以上

显存：NvidiaRtx2060 8G及以上,Rtx3060,3070,4060,4070,4080 16G以上

CUDA:CMD->nvidia-smi CUDA Version: 12.2+

硬盘：100G+的固态放模型和LM Studio

## 三、安装设置
先去官网地址下载对应平台的 LM Studio
[LM Studio Discover, download, and run local LLMs](https://lmstudio.ai/)

![](/assets/images/技术/AIGC/大模型/LLM/pic1.png)

下载完成后，不需要安装，双击就直接打开了。

首次打开，并没有大语言模型，需要自己下载模型之后才能使用，需要注意的是，默认模型下载地址是在 C 盘的，如果你的 C 盘空间吃紧，建议修改到其他路径。修改方式如下：

![](/assets/images/技术/AIGC/大模型/LLM/pic2.png)

**换源（optional）**
这个可选的，如果你不会魔法上网，则需要这一步换源。
在图标处，右键 -> 打开文件所在位置。
`app-x.x.xx/resources/app/.webpack/`

>resources/app/.webpack/main/index.js
resources/app/.webpack/main/llmworker.js (0.2.23 及以后版本是llmworker了，之前 unity.js)
resources/app/.webpack/main/worker.js
resources/app/.webpack/renderer/main/main_window/index.js

复制备份这几个文件，把其中所有的 huggingface.co 都替换成 hf-mirror.com
然后保存就行。

## 四、下载模型并运行
下载模型，比如下载阿里的通义千问

![](/assets/images/技术/AIGC/大模型/LLM/pic3.png)

一般会有很多版本，参数量不同，下载的时候根据自己的电脑配置进行选择。

使用进入 AI Chat 页面。选择一个即可。

![](/assets/images/技术/AIGC/大模型/LLM/pic4.png)

## 五、配置 open-webui
如果你只是自己使用，上面的已经够了。
如果还想让别人一起使用，并且爱折腾，则可以搭配 open-webui ，用网页的形式使用。
关于 open-webui 安装也很简单，方式有很多，比如使用 docker 或者手动安装。这里我采用手动安装方式。

1. 你需要有 python 3.11 的环境，然后通过 pip 安装。

```
pip install open-webui
```

2. 打开 web 界面。

```
open-webui serve
```

当你看到如下界面，说明成功了。

![](/assets/images/技术/AIGC/大模型/LLM/pic5.png)

然后打开网址： [http://localhost:8080/](http://localhost:8080/)

正常情况下是没有问题的，如果你看到如下类似的错误页面：

![](/assets/images/技术/AIGC/大模型/LLM/pic6.png)

则再次手动输入地址 [http://127.0.0.1:8080/](http://127.0.0.1:8080/)

![](/assets/images/技术/AIGC/大模型/LLM/pic7.png)

看到如上的页面，说明 open-webui 安装启动成功了。
接下来注册账号，登录。

**配置 LM Studio 和 Open-Webui**

在 LocalSever 中以 chat 方式启动 LM Studio 对话。

![](/assets/images/技术/AIGC/大模型/LLM/pic8.png)

看到下面的额日志则表示启动成功。复制 ⑤ 中的 url，然后打开 open-webui 的网页。依次点击右上角 `设置 -> 管理员设置 -> 外部链接` 。 将复制的 url 配置上去，最后记得保存。

![](/assets/images/技术/AIGC/大模型/LLM/pic9.png)

接下来回到对话页面，就可以愉快的使用了。

同一局域网的用户则可以通过你的ip地址进行访问。

## 写在结尾
学习 AIGC 已经很久了。这是我写的第一篇文章，写的非常详细，旨在小白用户也能搭配好大语言模型的本地环境。然后用起来，提升工作效率。后续会写更多 AIGC 应用相关的文章。