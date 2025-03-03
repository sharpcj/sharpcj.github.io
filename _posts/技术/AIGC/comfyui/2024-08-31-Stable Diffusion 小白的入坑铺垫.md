---
layout: post
title: "Stable Diffusion 小白的入坑铺垫"
date:  2024-08-31 17:58:12 +0800
categories: ["技术", "AIGC", "comfyui"]
tag: ["AIGC", "Stable Diffusion"]
image:
  path: /assets/images/技术/AIGC/comfyui/sd 入门/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: stable diffusion 绘图基础
---

小白的 Stable Diffusion 入坑铺垫

本文主要讲述一些 Stable Diffusion 入坑前需要了解的一些相关概念，不会涉及很高深的理论知识，因为我也讲不明白。本文所讲的内容基本上小学生就能看懂。如果你完全没听说过 Stable Diffusion 也没关系，只要你听说过 AI 绘画，并且对此有兴趣，就能跟着我一步步了解入坑。如果你想更进一步了解更深层次的计数原理，本文后面会给出一些连接，都是我看过的比较不错的文章或者视频。

## 一、AIGC 的概念
2022年，是人工智能爆发的元年，前有 Stability.Ai 公司开源了 Stable Diffusion 模型，后有 Open AI 发布了 ChatGPT，二者都是 AI 领域发展的里程碑式的事件。它们让 AI 不再是科研学术领域专属的高深莫测的技术名词，而是真真实实让普通人触手可及，提高生产效率的智能工具。
那 AIGC 是什么呢，AIGC （Artificial Intelligence Generative Content），即人工智能生成内容。这个领域的比较宽泛，生成的内容可以是文本，图像，音频，视频等等。机器可以跟人一样，能够看到、听到、思考、判断，然后做出决策，生成上述内容。比如前面提到的 ChatGPT 就是 AIGC 领域的一个具体应用。
本文接下来将围绕 Stable Diffusion 来介绍。

## 二、Stable Diffusion
Stable Diffusion, 潜在的扩散模型，是一种深度学习文本到图像生成模型，它主要根据文本描述生成图像。简单来说是一种文生图的算法。由 Stability.Ai 开源。

### Stable Diffusion 和 Midjourney

![](/assets/images/技术/AIGC/comfyui/sd%20入门/pic1.png)

目前市面上比较权威，并且能真正用于工作中的 AI 绘画软件，其实就两款，一个是 Midjourney（简称MJ），另一个就是 Stable Diffusion（简称 SD），MJ 需要付费使用，使用起来相对简单。而SD开源免费，但是上手难度和学习成本略大，并且对电脑配置有一定要求。

两者在实际使用中也各有利弊，从大的方面来讲，MJ 在生图图片时更具想象力，生成图片的在细节上略优于 SD,商业服务完善，助力艺术创作。SD 比 MJ 拥有更加丰富的个性化体验，使用者可以进行更精细的调教，以此生成更贴近需求的图片。得益于 SD 的开源，全世界的开发者和爱好者都可以参与进来，SD 拥有非常活跃的社区，非常丰富好用的自定义插件，甚至 SD 在 AI 生成视频特效、音乐生成等领域也有所建树。

## 三、Stable Diffusion 对电脑配置的要求
电脑配置最核心的配件，是 CPU、显卡、内存、硬盘。一般在 AIGC 领域，最重要的还要数显卡，很多 AI 应用只支持 N 卡（英伟达 Nvidia 独立显卡）。使用 Stable Diffusion 最常用的两种方式有两种 webui 和 comfyui 。其中 webui 对电脑显卡的要求最低 10 系起步，体验感佳 40 系。其中显存大小也很重要，最低 4G， 6G 及格，内存最低 8G, 16G 及格，硬盘空间最好有 500G 以上，固态硬盘最佳。而如果使用 comfyui，则对电脑配置要求更低，最低 3G 显存可用，出图速度也更快。

重要的事强调一遍：显卡最重要，尽量选 N 卡，支持 Cuda，显存也重要。显卡计算能力强弱，只是出图时间长短的问题，显存不够，直接就玩不了。

![](/assets/images/技术/AIGC/comfyui/sd%20入门/pic2.jpg)

详细的数据对比，大家可以到各大论坛，或者 Nvidia 官网了解。

## 四、概念理解
我自己在学习过程中，经常看到有一些刚入门的小伙伴，问 Stable Diffusion 和 Comfyui 学哪个。实际上，这个问题本身就是错误的。提问的人没有分清楚一些基本概念。

前面讲到，Stable Diffusion 是一种扩散模型。常见的使用方法有 webui 和 comfyui 两种方式。
webui 使用界面如下:

![](/assets/images/技术/AIGC/comfyui/sd%20入门/pic3.jpg)

comfyui 使用界面如下：

![](/assets/images/技术/AIGC/comfyui/sd%20入门/pic4.png)

相比之下，webui 更适合新手入门，所有操作在界面上一目了然，上手起来很容易。而 comfyui 是工作流模式，需要添加各种节点，并将它们用线连起来，更符合 stable diffusion 的工作流流向，如果你对深入学习 stable diffusion 有兴趣，可以选择 comfyui，另外 comfyui 可以保存成 json 文件，用来复用，comfyui 生成的图片中默认也包含完整的工作流信息，可以将工作流 json 文件，或者由 comfyui 生成的图片直接拖入 comfyui 中，还原整个工作流。
webui 比较稳定了，迭代更新速度也较慢，而 comfyui 目前几乎每天都会有新版本。具体使用哪个，看个人意愿。
这里只要是澄清，无论是 webui 还是 comfyui 都是上层的应用形式，stable diffusion 只是一种模型。比如近期非常火爆的一种新的文生图模型 Flux，它也是可以在 webui 种运行。

## 五、 结尾放图
首先给出一些学习过程中我认为非常好的资料连接：
[7000字详解！幼儿园都能看懂的 Stable Diffusion 工作原理](https://www.uisdc.com/stable-diffusion-42)

[Stable Diffusion 维基百科](https://zh.wikipedia.org/zh-cn/Stable_Diffusion)

[B站秋葉大佬的视频](https://www.bilibili.com/video/BV1x8411m76H/?spm_id_from=333.999.0.0&vd_source=eefd9080674a3e446f95033c5f045e0c)

![](/assets/images/技术/AIGC/comfyui/sd%20入门/pic5.png)

![](/assets/images/技术/AIGC/comfyui/sd%20入门/pic6.png)

![](/assets/images/技术/AIGC/comfyui/sd%20入门/pic7.png)

目前来看，Stable Diffusion 能做的工作相当多，比如，模特换装，照片放大，局部重绘等等，感兴趣的朋友可以认真学习一下。