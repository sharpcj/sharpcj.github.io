---
layout: post
title: "Jetpack Compose(1) —— Android 全新的 UI 框架"
date:  2024-02-27 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

## 写在前面
Jetpack Compose 已经不是什么新技术了，Google 早在 2019 年就推出 Jetpack Compose 的首个 alpha 版本，时至今日，相当大比例的国内 Android 开发者还没有学习使用过。本篇文章主要介绍 Jetpack Compose 的一些发展背景，接下来会有一个系列的文章，来逐步讲述 Jetpack Compose 的一些知识点。

## 一、Jetpack Compose 是什么
### 1.1 全新的 Android UI 开发框架
Jetpack Compose 是用于构建原生 Android 界面的新工具包，是一种全新的声明式 UI 框架，它使用更少的代码、更直观的 Kotlin API 以及拥有更强大的开发工具，可以帮助开发者加快 Android 开发界面的开发。你可以理解为是传统的 View 视图体系的替代品。

**关键词： 原生界面开发、 Kotlin 语言、 声明式 UI。**

### 1.2 命令式UI 与 声明式UI
传统的 View 视图体系属于命令式的编程范式，什么是命令式？就是我们需要用命令的方式告诉计算机如何去做事情。比如先获取到一个 view 的对象的引用，再通过调用它的 setXXX() 方法去改变这个 view 的属性，以此来更改 UI 状态。手动操纵视图会提高出错的可能性。如果一条数据在多个位置呈现，很容易忘记更新显示它的某个视图。此外，当两项更新以出人意料的方式发生冲突时，也很容易造成异常状态。例如，某项更新可能会尝试设置刚刚从界面中移除的节点的值。一般来说，软件维护的复杂性会随着需要更新的视图数量而增长。

前面提到，Jetpack Compose 是声明式 UI 框架，对于声明式 UI 框架，我们只需要在描述一个页面的时候附带上各个控件的状态，然后当有任何状态发生改变时，页面会自动进行刷新。初次了解声明式 UI 的人可能会有疑问，这样为了更新页面中的某个控件的，去刷新整个页面，那不是很低效，页面复杂起来了，不是要卡爆？ 事实上，所有的声明式 UI 框架都采用了相似的优化策略，再刷新页面时，只会更新哪些状态有变化的控件，而那些状态没有变化的控件会跳过执行。在 Jetpack Compose 中状态改变，页面进行刷新，称之为重组，状态没有变化的控件则会跳过重组，关于重组和跳过重组，会在后面的文章中详细讲述。

下面举个实际的例子：
考虑一个带有未读邮件图标的电子邮件应用程序。如果没有消息，应用程序将呈现一个空白信封。如果有一些消息，我们在信封中渲染一些纸，如果有100条消息，我们将图标渲染为着火的样子。。

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic1.jpg)

对于命令式接口，我们可能需要编写这样的更新计数函数:

```
fun updateCount(count: Int) {
  if (count > 0 && !hasBadge()) {
    addBadge()
  } else if (count == 0 && hasBadge()) {
    removeBadge()
  }
  if (count > 99 && !hasFire()) {
    addFire()
    setBadgeText("99+")
  } else if (count <= 99 && hasFire()) {
    removeFire()
  }
  if (count > 0 && !hasPaper()) {
   addPaper()
  } else if (count == 0 && hasPaper()) {
   removePaper()
  }
  if (count <= 99) {
    setBadgeText("$count")
  }
}
```

在这段代码中，我们收到了新的计数，必须弄清楚如何更新当前 UI 以反映该状态。这里有很多边界情况，尽管这是一个相对简单的例子，但这种逻辑并不容易。
那在声明式接口中，如何编写呢？

```
@Composable
fun BadgedEnvelope(count: Int) {
  Envelope(fire=count > 99, paper=count > 0) {
    if (count > 0) {
      Badge(text="$count")
    }
  }
}
```

>在这里我们说：
如果计数超过99，显示火力。
如果计数超过0，显示纸张，
如果计数超过0，则渲染计数徽章。

这就是声明性API的含义。我们编写的代码描述了我们想要的 UI，但没有描述如何转换到那种状态。这里的关键是，在编写这样的声明性代码时，您不再需要担心UI的上一个状态，只需要指定当前状态。框架控制如何从一个状态转换到另一个状态。因此，我们不再需要考虑它。

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic2.png)

其实，在 Jetpack Compose 之前，Google 推出的 Data Binding 框架也属于声明式编程范式。

**小结：**命令式关注过程，声明式关注状态。

## 二、Google 为什么力推 Jetpack Compose
View 视图体系自 Android 诞生起，一路发展至今，为何 Google 还要推出 Jetpack Compose 呢？

### 2.1 开发效率更高
传统的 Android UI 开发采用的是 View 视图体系，它是一种命令式的编程模型。开发者需要通过编写大量的代码来描述 View 的外观和行为，包括布局、样式、交互等。这种方式需要开发者花费大量的时间和精力，而且代码量也非常庞大，维护和扩展也比较困难。

与之相比，Jetpack Compose 是一种声明式的编程模型，采用的是函数式编程的思想。开发者只需要描述 UI 的样式和行为，然后 Jetpack Compose 会自动处理 View 的布局和绘制。这种方式不仅减少了代码量，还可以提高开发效率，同时也使得代码更加易于维护和扩展。

那 Data Binding 呢？用过的人自然能明白其中的痛苦，在 xml 文件中编写代码。并且还需要先解析 xml 文件，效率自然会有影响。而 Jetpack Compose 代码完全基于 Kotlin DSL 实现，相较于 Data Binding 需要有 xml 的依赖，Compose 只依赖单一语言，避免了因为语言间交互而带来的性能开销及安全问题。

### 2.2 组合优于继承
组合优于继承，是面向对象设计模式中反复强调的原则。而现实是，继承用起来很方便，很多时候，大家很难从组合的视角思考问题。然而在复杂的体系中，继承关系往往很难梳理清楚。
传统的 View 视图是基于继承的结构体系，也已经随着 Android 发展了十多年。View.java 本身也变得十分臃肿，目前已经超过三万行。臃肿的父类控件，也会造成子类视图的功能不合理。以 Button 为例，为了能让 Button 具备显示文字的功能，Button 被设计成了继承自 TextView 的子类。这样有很多不适用于按钮的功能也被继承下来了，TextView 的剪贴板功能，对 Button 来说显得不合理。而且随着 TextView 自身功能的迭代，Button 可能引入更多不必要的功能。
另一方面，想 Button 这类基础控件，只能跟随系统升级而更新，及使发现了问题，也得不到及时的修复。这也是为什么如今很多新的视图都以 Jetpack 扩展库的形式单独发布的原因。
而 Jetpack Compose 则是作为函数，相互之间没有继承关系。让开发这更多的是以组合的思想去思考问题。

那在 Compose 中 Button 如何展示呢？

```
Button(
    onClick = { /* 处理点击事件 */ },
    modifier = Modifier.wrapContentSize()
) {
    Text("I'm a button")
}
```

作为按钮，它本身的职责是提供点击事件，至于他内部的样式，则需要和对应的其它可组合函数(暂且称之为控件)进行组合。

## 三、为什么要学习 Jetpack Compose
这里，我不再提 Jetpack Compose 相对于 View 视图体系的优势。而是想从行业背景，领域发展角度述说。

### 3.1 声明式 UI 的编程范式发展的趋势
从编程范式的维度来说，声明式 UI 相对传统的命令式 UI，更加先进， React， iOS 的 SwiftUI， flutter， Jetpack Compose， 声明式 UI 开发思想已经席卷了整个前端开发领域。如果你还在使用传统 View 视图体系开发 Android 原生界面，那你应该尝试声明式 UI。

### 3.2 Google 确定的 Android 官方推荐的 UI 框架
如同 Kotlin 取代 Java 成为 Google 推荐的 Android 官方推荐的编程语言一样，Jetpack Compose 也是 Google 推荐的 Android 官方 UI 框架。并且，新版的 Android Studio 中集成了大量的特性来支持 Jetpack Compose 更高效的开发。这表明 Jetpack Compose 在未来的 Android 开发中将扮演重要的角色。目前来看，Jetpack Compose 并不会完全取代传统的 View 视图体系，而是会和传统 View 视图体系共存， 但是 Jetpack Compose 在未来会能成为 Android 开发的主流。从 Android 14 Framework 中我们能看到 Compose 的影子了。在 2018 年，SystemUI 模块中，我们能看到 Kotlin 改写的部分模块，在 Android 14 中，我们再 Settings 中再次看到了 Kotlin 的代码了。并且 Google 新建了一个 spa 包，这个包里面尝试把 “应用管理”、“应用管理通知”、“使用情况” 页面用 Jetpack Compose 重写了。

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic3.jpg)

除此之外，还有 3 个主页面也进行了重写，

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic4.png)

出于稳健考虑，Google 没有默认使能这些页面。如果你想提前吃螃蟹，可以使用以下命令来开启： adb shell settings put global settings_enable_spa true 通过搜索代码，也可以轻松找到这个开关在哪里被使用：

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic5.png)

我们可以去到相关页面看一下前后对比：

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic6.png)

由于Jetpack Compose的各个组件一直都在很好地践行Material Design 3，可以看到后者的视觉还原要更好，也不会出现前者那样MenuItem背景颜色不正确的问题。

至此，可能连 framework 开发人员都要开始学习 Jetpack Compose 了，应用开发还有什么理由不学习呢？

## 四、Jetpack Compose 与 View 视图体系对比
Compose 重新定义了 Android UI 开发方式，大幅提升了开发效率，主要体现在：

1. 声明式 UI， 不需要手动刷新数据
2. 取消 XML, 完全解除了混合写法的(XML + java、Kotlin) 的局限性，以及性能开销
3. 超强兼容性，大多数 jetpack 库(如 Navigation、ViewModel) 以及 Kotlin 协程都适用于 Compose, Compose 能够与现有 View 体系并存，你可以为一个既有项目引入 Compose。
4. 加速开发，更简单的动画和触摸事件Api, Compose 为我们提供了很多开箱即用的 Material 组件，如果的APP是使用的material设计的话，那么使用Jetpack Compose 能让你节省不少精力。
5. 精简代码数量，减少 bug 的出现
6. 强大的预览功能， Compose 预览机制可以实时预览动画，交互式预览，真正所见即所得。

### 4.1 Jetpack Compose 使用前后对比
官网示例： Tivi [Jetpack Compose 使用前后对比](https://zhuanlan.zhihu.com/p/386826633)

#### 4.1.1 APK 尺寸缩减

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic7.png)

△ 展示 Tivi APK 大小的图表

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic8.png)

△ 展示 Tivi 方法数的图表

在将迁移后的应用与接入 Compose 前的应用做比较后，我们发现 APK 大小缩减了 41%，方法数减少了 17%。

#### 4.1.2 代码行数
尽管比较软件项目时，计算源代码行数不是特别有用的统计方式；但这种方式能够提供一个视角，帮助我们了解事物是如何变化的。
使用了 cloc 工具。使用下面的命令可以排除各种构建文件、自动生成文件以及配置文件。

```
cloc . --exclude-dir=build,.idea,schemas
```

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic9.jpg)

△ 展示 Tivi 源代码行数的图表

毫不意外的，XML 行数大幅减少了 76%。再见了，布局文件，以及 styles、theme 等其他的 XML 文件 。

有趣的是，Kotlin 代码的总行数也下降了。我对此现象的理解是，现在应用中的模板代码减少了，同时我们也得以移除大量的视图辅助类和工具类代码。您可以看到，我在 这个 PR 中删除了多年来编写的近 3,000 行代码。

#### 4.1.3 构建速度

![](/assets/images/技术/编程/Jetpack%20Compose/compose1/pic10.png)

△ 展示 Tivi 构建时间中位数的图表

考虑到 Kotlin 编译器与 Compose 编译器插件为我们所做的事情，如位置记忆化、细粒度重组等工作，构建时间能够 减少 29%， 可以说十分惊人。

## 写在最后
Jetpack Compose 虽然是 Google 推出的 UI 框架，但它也不仅仅是 Android UI 框架。Compose 从设计的角度可以分为四层， 每一层都可以被单独使用。

- Material 位于最上层，基于 Material Design 系统实现了各种 Composable，同时提供了基于 Material Design 的主题、图标等。
- Foundation 模块提供了一些基础的 Composable, 例如 Row、Column、LazColumn 等布局类 UI,以及特定的手势识别等，这些基础 Composable 可以在不同的平台通用。
- UI 层的功能众多，包含多个模块，是上层 Composable 运行的基础，例如 Composable 的测量、布局、绘制、时间处理以及 Modifier 管理等。
- Runtime 层实现了基本的 UI 树的管理能力，支撑了 Compose 通过 UI 树的 diff 驱动界面刷新的功能。如果只需要 Compose 树的管理功能，不需要 UI, 则可以直接基于此层进行构建。
其中 Runtime 层是核心，上层 UI 只是其应用场景之一。

目前 Jetbrains 公司已经基于 Jetpack Compose，启动了 Compose Multiplatform 项目，未来，Compose 可能是跨平台方案的选择之一。作为 Jetpack Compose 系列开篇文章，本文只是抛砖引玉。接下来如果有兴趣继续学习下去，可是要下一番大功夫了。