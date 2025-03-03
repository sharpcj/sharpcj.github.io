---
layout: post
title: "ComfyUI 基础教程(三) —— 应用 Controlnet 精准控制图像生成"
date:  2024-09-07 17:58:12 +0800
categories: ["技术", "AIGC", "comfyui"]
tag: ["AIGC", "Stable Diffusion", "comfyui"]
image:
  path: /assets/images/技术/AIGC/comfyui/comfyui3/default.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: stable diffusion 绘图基础
---

## 一、前言
你是否有见过下面类似这样的图片：

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic1.jpg)

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic2.png)

看起来平平无奇，当你站远点看，或者把眼睛眯成一条缝了看，你会发现，这个图中藏有一些特别的元素。这就是利用了 Ai 绘画中的 ControlNet，实现对图片的相对更精准控制。
上一篇文章讲述了文生图的基本工作流和最基础的核心插件用法。通过提示器可以描述我们想要生成的图片。但是通过文字是无法精准描述描述图片的。就好你说 “一个女孩，穿着粉色的裙子”，一百个人听到这句话，脑海中产生的信息有一百种，每个人想到的都不一样，无论你再怎么加场景、细节、修饰符，都不可能统一所有人的理解。那今天要将的 ControlNet 却能在一定程度上指导 Stable Diffusion 图片的生成过程，实现一些特殊的效果。

## 二、ControlNet 的相关概念
### 2.1 什么是 ControlNet
ControlNet 是一个控制预训练图像扩散模型（例如 Stable Diffusion）的神经网络。它允许输入调节图像，然后使用调节图像来控制图像生成。这里的调节图像类型有很多，比如涂鸦、边缘图、姿势关键点、深度图、法线图、分割图等。这些输入条件都可以作为条件输入来指导生成图像的内容。

这里需要明确一点，ControlNet 是一种算法，用来控制预训练图像扩散模型生成图像的，可以搭配 Stable Diffusion 模型进行使用，但不是只能使用在 Stable Diffusion 中。

### 2.2 ControlNet 的使用场景
ControlNet 的使用场景非常之多。比如想要控制生成图像人物的姿势，可以给定一张参考图，使用姿势控制模型，提取出人物的姿势，进行控制；比如生成室内装修设计图，可以根据给定定一个法线图，进行控制；比如可以简单画个涂鸦草图，让 AI 根据这个草图进行绘制等等。由于 AI 绘图试一项创造性工作，所以无法列举完所有的场景，这里大家有个初步印象，然后可以尝试不同的 ControlNet 模型，发挥想象，用到合适的需求场景。

### 2.3 ControlNet 官方地址
ControlNet 的官方地址：[https://github.com/lllyasviel/ControlNet](https://github.com/lllyasviel/ControlNet)

关于 ControlNet 的原理，官网上有一些讲解。本人水平有限，这里就班门弄斧，以免误导大家。感兴趣的朋友可以查看官网文档，或者搜索其它资料文献。

## 三、如何使用 ControlNet
对于使用 AI 绘画的人来说，应用才是最重要的。接下来通过一个示例讲解如何使用 ControlNet。

### 3.1 安装 ControlNet 插件
**ComfyUI-Advanced-ControlNet**
首先是 ControlNet 节点安装，如果你使用的秋葉整合包 ComfyUI，是自带了 ControlNet 节点的。如果没有，可在自行安装插件 `ComfyUI-Advanced-ControlNet`。相信能看到这里的朋友都掌握了插件安装的方法，如果有不会的朋友，可以回头去看我前面的文章，安装插件的方法前面的文章有讲。

### 3.2 模型下载
强调： ContrlNet 模型是分版本的，与基础大模型要对应。 比如你基础大模型使用的 SD1.5, 那么你的 ControlNet 模型也要选择 SD1.5 版本,模型不匹配运行会报错。

官方提供了 Stable Diffusion 1.5 版本的 ControlNet 模型下载地址：
[https://huggingface.co/lllyasviel/ControlNet-v1-1/tree/main](https://huggingface.co/lllyasviel/ControlNet-v1-1/tree/main)

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic3.png)

这里面的模型非常之多，通过模型的名字基本能知道模型的作用。

下载模型，放到如下路径：`ComfyUI\models\controlnet`

如果需要下载 SDXL 或者其它版本的 ControlNet 模型，可自行在模型下载网站搜索下载。

#### 多合一模型
`controlnet-union-sdxl-1.0`
这个模型只适用于 SDXL 版本，优势是该模型集成了 ControlNet 的 12 中类型的模型，不用挨个下载, 并且能更节约内存。
官方地址：[https://huggingface.co/xinsir/controlnet-union-sdxl-1.0](https://huggingface.co/xinsir/controlnet-union-sdxl-1.0)

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic4.png)

### 3.3 ControlNet 使用
接下来通过一个实际的例子，展示 ControlNet 如何使用。

#### 3.3.1 应用 ControlNet 节点
首先，我们不必要完全从零开始搭建工作流，直接加载默认的文生图工作流，在它的基础上进行修改。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic6.png)

然后我们添加一个 应用ControlNet 节点。
`新建 -> 条件 -> ControlNet` 这里面可能有好几个相关的节点。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic5.png)

这里说说最常用的 `ControlNet应用高级` 这个节点。掌握这个节点，其它节点使用方式大同小异，大家可以自行尝试。
`ControlNet应用高级` 和 `ControlNet应用` 相比，多了负面条件控制，以及开始时间和结束时间。我们可以把两者都添加进来更直观的看到。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic7.png)

这里输入和输出中的正面条件和负面条件都很好理解，说一下开始时间和结束时间，它们指的是，在整个采样器去除噪波，生成图片过程中，ControlNet 介入控制的时机，最小是0，最大是1。如果开始时间设置为0，结束时间设置为1，则表示生成图片过程中，ControlNet 从头到尾都介入控制。

再看剩余的参数和输入条件：

- **强度：**是 ControlNet 的作用强度。值越大，生成的图片 ControlNet 参考强度越大，也越贴近参考效果，但是实际使用中，往往也不是参考效果越明显越好。比如前言中看到的光影文字效果，我们以文字的深度图作为 ControlNet 参考图，生成风景照，当 ControlNet 强度太大了，生成的图像中，文字非常明显， 反而风景照发挥的空间太小了，往往效果不近人如意。实际使用过程中，还需要根据实际生成图片不断调整。
- **ControlNet：**该输入项需要连接到 ControlNet 加载器。前面我们下载的模型，需要根据需要选择一个使用。
图像：这里需要一个输入一个经过 ControlNet 预处理器处理过的图像。

#### 3.3.2 ControlNet 模型加载器
ControlNet 模型加载器同样有很多种，这里挑一个最常用的。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic8.png)

只有一个参数，选择我们需要的模型。前面强调过，这里的模型版本，要和你选择的基础大模型匹配。

#### 3.3.3 ControlNet 预处理器

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic9.png)

**ControlNet 的预处理器种类非常多，选择哪个需要和加载的 ControlNet 模型对应上**。 比如前面选择了模型加载了 `openpose` 类型，这里就要选择面部与姿态下面的预处理器。

这里我们选择 DW姿态预处理器来说明。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic10.png)

这里面的参数，检测项需要就启用，不需要就禁用。分辨率可以根据基础大模型设置。一般 SD1.5 设置 512, SDXL 设置 1024。BBox检测和姿态预估是用来检测身体各个部位的，在其他场景中也会用到，不同模型之间的区别，大家可以自行了解。

首次使用某个 ControlNet 预处理器时间会长一点，会自动从网络下载所需要的模型，请保持网络通畅，并且可以在 ComfyUI 启动器的控制台看到下载任务和进度。后续如果本地有了该模型，很快就可以加载进来。

输入图像，需要出入一张参考图像。

**Aux集成预处理器**

下面再介绍一个多合一的 ControlNet 预处理器 —— Aux 集成预处理器。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic11.png)

它只有两个参数，预处理器，需要自己选择，比如选择 DW姿态预处理，就和上面的效果基本一致了。同样需要输入一张参考图。

**图像加载节点**

输入的参考图，就是从本地或者网络加载一张图片，然后输出到 ControlNet 预处理器。这里就需要使用到图像加载节点。

- 加载本地图像
加载图像使用 `LoadImage` 节点就可以。`新建->图像->加载图像`

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic12.png)

点击 `choose file to upload` ， 选择一张本地图片，或者直接拖动图片到上面就可以，实际使用中，拖拽一张网络图片也是可以的。

- 加载网络图像
有时候我们知道网络图片的地址，那就可以使用专门加载网络图片的节点。我们可以搜索一下本地节点，没有的话，就需要自己安装。我这里有很多插件提供了加载网络图片的节点。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic13.png)

以上就是所有 ControlNet 需要的节点，都添加进来然后正确连接起来就可以了。

### 3.4 实例展示
#### 3.4.1 人物姿态控制
这里本地有一张图片

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic14.jpg)

下面生成一张女孩的图片，姿态和这张图片保持一致。工作流如下：

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic15.png)

和上面讲解的节点一样，这里，注意就是 Clip 文本编码器输出的正面条件和负面条件连接到 ControlNet 应用节点的输入，ControlNet 应用节点的输出再分别连接到 K 采样器的输入。
另外这里将 ControlNet 的预处理器输出到一个预览图像节点，方便我们查看与预处理处理后的结果。

大模型选择的 `xxmix9Realistic` 写实风格，正向提示词就简单填写的 `1 girl`, 最终的结果生成了一个女孩，姿势与我们的参考图是一致的。当然这个图并不完美，放大看比较模糊，细节也很欠缺。如果要生成一张质量较高的图片，需要非常好的提示词，好需要一些其他的节点。如增加 Lora 模型(后面文章讲解如何使用)，放大处理节点，对局部进细节进行处理（比如手部修复，面部修复）等等。

#### 3.4.2 局部重绘
局部重绘的方法有很多，本示例将通过 ControlNet 完成一个局部重绘的简单工作流。
想要实现的效果是对下面这张图片进行局部重绘，比如给这个女孩换上不同的裤子。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic16.png)

先说一下基本的思路，就是加载上面的图像，然后创建需要重绘部位的遮罩（在 PS 里面也称为蒙版），然后对遮罩区域进行重新绘制，这里就要创建图中女孩裤子部分的遮罩。此时，使用 ControlNet 的模型是 `inpaint` 类型。针对这个示例，接下来使用 SDXL 版本的模型进行，一步一步实现效果。
先上图：

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic17.png)

和前面使用 openpose 的工作流基本类似。这里 SDXL 的模型，我选择的是 union 版本，这里在加载 ControlNet 模型的时候，ControlNet 加载器不能直接连接到 应用 ControlNet 节点上，因为我们使用 union 版本的模型，它集成了12种类型的模型功能，使用时就需要选择具体使用哪种类型，这应该很好理解。
所以这里连接到 设置 `UnionControlNet` 类型 这个节点，然后再将输出端连接到 应用ControlNet 节点。

在设置 `UnionControlNet` 类型中，基于重绘，我们选择 repaint 类型。同时，与之对应的预处理器节点应该选择 `Inpaint` 内部预处理器。该预处理需要输入图像和遮罩。刚刚也说过了，我们需要创建遮罩区域，对遮罩区域进行重绘。

如何创建图像遮罩呢？
在图片上鼠标右键，点击"在遮罩编辑器中打开"，

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic18.png)

然后会自动打开遮罩编辑器，操作非常简单，左下角工具条依次是清除遮罩、设置画笔宽度、设置遮罩透明度、设置遮罩颜色。如果涂错了，点击清除，或按住右键涂抹即可。最后记得点击右下角保存。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic19.png)

另外记得大模型也要选择对应的 SDXL 版本，对于 SDXL 分辨率可以设置为 1024 左右。正向提示词写上“1 girl,short red_dress, ”。最后运行看看：

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic20.png)

不出意外的话，意外出现了。。。生成的图像一片漆黑，预览预处理器处理过后的图像倒是能看清楚图像和遮罩。
不要慌，这里，有个小细节，就是 inpaint 内部预处理处理过后的图像不能直接输入到 ControlNet 应用节点，需要先转换为 RGB。

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic21.png)

再次运行，可以看到，图片中的人物已经换上了红色的裙子。
Ok，这次没问题了。

**注意这个图像转RGB节点，是 WAS 插件下的图像到RGB**。 不要添加为其他自定义插件中的 `Convert Image To RBG` 了，亲测过，不生效。

我们还可以再进一步补充完善完善一下这个工作流。上述这个工作流，K采样器的 Latent 我们传入的试一个空 Latent，图像的宽高是我们自己设置的，这样有两个特点：

1. 有可能我们设置的值与原图宽高不一致
2. 空的 Latent 意味着对整个图完全进行了重绘。关于这一点，后面在专门写一篇文章，讲述各种重绘方式的区别。

由于是图片局部重绘，我们实际上可以将原图使用 VAE编码器进行编码转换成 Latent 输出到采样器中。这里使用 VAE 内补编码器。输入端，需要传入图像，VAE 模型，遮罩，输出 Latent 连接到采样器。VAE内补编码器还有一个参数，“遮罩延展”，这个类似与 PS 中的羽化，可以让重绘部分和原图融合得更好。这个值可以在使用过程中不断修改尝试，达到自己满意的效果就可以了。

最后完整的工作流如下：

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic22.png)

### 3.5 ControlNet 堆的使用
有时候我们想用多张图片的不同 ControlNet 类型进行控制，最原始的做法就是依次添加不同的 ControlNet 相关的节点，把它们依次串联起来。这样做节点数看起来会很多，不是很直观。ComfyUI 中可以使用 ControlNet 堆进行统一处理。使用也非常简单：

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic23.png)

如果你没有这个自定义插件，就需要额外下载，看这个节点样式，使用起来非常简单。它支持最多三个 ControlNet 一起控制，整合了 ControlNet 模型加载器，并且分别提供了开关、强度、介入时间等参数。如果 3 个不够用，还可以继续串联 ControlNet 堆节点。一般来说。

## 四、结束语
ControlNet 在 AI 绘画中非常重要，大多数生产环境中，都不是天马行空，完全随机生成图片的，有了 ControlNet, 能够更精准地控制生图，才能不断修改完善，符合实际使用需求。本文只是讲解了 ControlNet 的入门使用方法，更多的知识需要在大量的练习和探索中掌握。
回到开头图片，文字融入图片的效果又该如何应用 ControlNet 来生成呢？
留给大家自己去学习探索。最后中秋节快到了，提前祝大家中秋节快乐！

![](/assets/images/技术/AIGC/comfyui/comfyui3/pic24.png)