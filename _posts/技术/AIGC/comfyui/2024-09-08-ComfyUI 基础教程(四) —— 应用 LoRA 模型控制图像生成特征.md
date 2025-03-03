---
layout: post
title: "ComfyUI 基础教程(四) —— 应用 LoRA 模型控制图像生成特征"
date:  2024-09-08 17:58:12 +0800
categories: ["技术", "AIGC", "ComfyUI"]
tag: ["AIGC", "Stable Diffusion", "comfyui"]
image:
  path: /assets/images/技术/AIGC/comfyui/comfyui4/default.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: stable diffusion 绘图基础
---

## 前言
上一篇文章讲述了 ControlNet 模型的使用，如果掌握了 ControlNet 的使用之后，本文所讲的 Lora 模型将会非常容易理解。

## 一、LoRA 模型的概念
之前的文章中，在介绍模型种类时，简单介绍过 Lora 模型，LoRA 模型全称是 Low-Rank- Adaptaion of Large Language Models。是一种用于微调大语言模型的低次序适应技术。它最初用于 NLP 领域，特别是用于微调 GPT-3 等模型。在 Stable Diffusion 模型应用中， Lora 允许用户在不修改 SD 大模型的情况下，利用少量数据训练出具有特定画风、IP 或者人物特征的模型，这种模型在社区和个人开发者非常受欢迎。

想深入了解 LoRA 的原理的朋友，请自行学习。

## 二、Lora 的效果展示
以上都是行话。下面用通俗的语言介绍。
LoRA 相当于大模型之外的另外一种模型，它的主要作用是给原有的画面做一些微调。LoRA的种类有很多，有人物的、动作的、建筑的，甚至改变一些画风的、表情的、物品的，只要你加了某个特定的LoRA，它就会在画面中呈现。最常用的一个就是人物的，举个例子，最近非常火的黑神话悟空的各种图片:

![](/assets/images/技术/AIGC/comfyui/comfyui4/pic1.png)

这就是使用悟空的 LoRA 生成的。

下面这个，用的是胡桃的LoRA

![](/assets/images/技术/AIGC/comfyui/comfyui4/pic2.png)

可以看到不管怎么切换基础模型，不管怎么切换画风，LoRA始终是胡桃的LoRA，它画出来的就是胡桃。

## 三、 Lora 的使用
### 3.1 Lora 模型加载器
前面学会了使用 ControlNet，在来看 Lora 的使用就非常简单了。
首先到模型网站上下载自己想要的 Lora 模型。放到 `ComfyUI\models\loras` 文件夹下。然后添加一个 Lora 加载器节点。

![](/assets/images/技术/AIGC/comfyui/comfyui4/pic3.png)

该节点有三个参数，
- 选择 Lora 模型，
- Lora 模型的强度 - 这个数值越大，生成图像月偏向 Lora 模型风格
- 以及 Clip 的强度 - 文本提示词的强度

这些参数根据需要调整就行。

最后再将主模型的模型输出点连接到 Lora 加载器的输入点，主模型的 Clip 输出作为 Lora 加载器的输入，最后，Lora 加载器的输出，分别连接到 Clip 文本编码器和采样器的模型输入。

需要强调的是：**Lora 模型的版本要和主模型版本保持一致**。

### 3.2 Lora 使用演示

![](/assets/images/技术/AIGC/comfyui/comfyui4/pic4.png)

可以看到，当前选择了一个中国风的水墨场景的 Lora ，生成的图像，也就成了中国风的水墨风格。

### 3.3 Lora 堆的使用
如果要使用多个 Lora 模型，可以添加多个 Lora 加载器，依次串联起来。和 ControlNet 一样， 多个 Lora 模型可以使用 Lora 堆节点进行加载。
Lora 堆节点有很多，有的统一控制 Clip 强度，有的分别增加了开关，可以对每个 Lora 分别控制，但是基本都大同小异。这里展示简易 Lora 堆节点。

![](/assets/images/技术/AIGC/comfyui/comfyui4/pic5.png)

这里有一个总开关，可以开启或者关闭应用 Lora 堆，模式高级的化可以针对每个 Lora 模型设置 Clip 权重，最后输出到 `应用LoRA 堆` 节点。

## 四、本地训练 LoRA 模型
前面提到，Lora 有一个特点，就是个人消费级显卡也可以很快完成训练，这样就给了我们很多施展的空间。
下面来讲解训练 LoRA 模型的步骤：

### 4.1 下载安装 LoRA 训练器
训练模型的过程，俗称“炼丹”，训练模型的软件和环境，称为模型训练器，俗称“丹炉”。训练模型首先需要有一个模型训练器，目前网上有很多版本的 SD LoRA 模型训练器，这里还是比较推荐 秋葉 大佬的LoRA 模型训练器。
下载地址：
[https://pan.baidu.com/s/1TBaoLkdJVjk_gPpqbUzZFw](https://pan.baidu.com/s/1TBaoLkdJVjk_gPpqbUzZFw)
[https://pan.quark.cn/s/b4ccd1f635b6](https://pan.quark.cn/s/b4ccd1f635b6)

### 4.2 训练数据准备
训练数据准备，主要做好下面几点：

1. 训练素材处理
首先要准备好训练素材，比如你想训练某个特定的人脸 LoRA 模型，你需要准备 一些该特定人物的人脸的照片，素材的质量直接决定了训练出的模型的质量。对素材有一些的要求，比如图片数量不少于 15 张，一般可以准备 20-50 张；图片的质量要高，主体内容清晰可见，特征明显，图片尽量构图简单，避免其它杂乱的元素；尽可能以人物脸部特写为主，多角度，多表情等; 减少重复或者相似度太高的图片。
素材准备完毕后，还需要对图片进一步处理，对低像素的素材图片，进行高清放大处理。图片统一分辨率，注意分辨率为 64 的倍数，显存低的可切为 512 x 512, 显存高的可切为 1024 x 1024。可以借助一些软件或者工具批量裁切。

2. 素材打标
这一步的关键是对素材进行打标。

4. 打标优化
打标完成之后，还要对生成的文件中的标签进行再优化，删除某些和要训练的风格不一致的图片。
这一步相对比较复杂一点，训练不同风格的 LoRA，打标的要求不一致。大家也可以在网上学习其它优秀的教程。

### 4.3 训练环境参数配置
这一步是需要配置训练的具体参数，比如每张素材图片训练多少步，总共训练多少轮等等。

### 4.4 模型训练
训练参数设置保存完之后，就可以开始训练了。训练过程中会自动进行模型保存。

### 4.5 模型测试
之前的文章有提到过，模型训练什么时候叫训练好了，是没有具体标准的，我们需要对训练生成的模型进行测试，感觉效果达到预期了，就可以停止训练了。

实操方式放上秋葉大佬视频
[秋葉大佬训练 LoRA 教程](https://www.bilibili.com/video/BV1AL411q7Ub/?spm_id_from=333.999.0.0&vd_source=eefd9080674a3e446f95033c5f045e0c)

## 五、结束语
LoRA 非常好用，使用也很简单，大家可以多到开源站点上看看别人训练的 LoRA，学习学习。本文相对比较简单，到此结束。