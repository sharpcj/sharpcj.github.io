---
layout: post
title: "Jetpack Compose(6)——动画"
date:  2024-04-26 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

本文介绍 Jetpack Compose 动画。
[官方文档](https://developer.android.com/develop/ui/compose/animation/introduction?hl=zh-cn)
关于动画这块，第一次看官网，觉得内容很杂，很难把握住整个框架结构，很难去对动画进行分类。参考了很多文献资料，大多数都是从高级别 API 开始讲解，包括官网也是如此。我发现这样不太容易理解，因为高级别 API 中可能会涉及到低级别 API 中的一些方法，术语等。所以本文从低级别 API 讲起。

## 一、低级别动画 API
### 1.1 animate*AsState
`animate*AsState` 函数是 Compose 动画中最常用的低级别 API 之一，它类似于传统 View 中的属性动画，你只需要提供结束值（或者目标值），API 就会从当前值到目标值开始动画。
看一个改变 Composable 组件大小的例子：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    // I'm here
    val boxSize by animateDpAsState(targetValue = if (bigBox) 200.dp else 50.dp, label = "")
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(boxSize) // I'm here
        .clickable {
            bigBox = !bigBox
        })
}
```

运行一下看看效果:

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic1.gif)

上述示例中我们使用了 `animateDpAsState` 这个函数，定义了一个 “Dp” 相关的动画。
其实 `animate*AsState` 并不是只某一个具体方法，而是只形如 `animate*AsState` 的一系列方法,具体如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic2_2.png)

是的，你没有看错，甚至可以使用 `animateColorAsState` 方法对颜色做动画。
聪明的你，肯定会有一个疑问，这个方法是从当前值到设定的目标值启动动画，但是动画具体执行过程是怎样的，比如持续时间等等,这个有办法控制吗？还是以 `AnimateDpAsState` 为例，看看这个参数的完整签名：

```
@Composable
fun animateDpAsState(
    targetValue: Dp,
    animationSpec: AnimationSpec<Dp> = dpDefaultSpring,
    label: String = "DpAnimation",
    finishedListener: ((Dp) -> Unit)? = null
): State<Dp> {
    return animateValueAsState(
        targetValue,
        Dp.VectorConverter,
        animationSpec,
        label = label,
        finishedListener = finishedListener
    )
}
```

实际上这个方法有4个参数。

- targetValue 是没有默认值的，表示动画的目标值。
- animationSpec 动画规格，这里有一个默认值，实际上就是这个参数决定了动画的执行逻辑。
- lable 这个参数是为了区别在 Android Studio 中进行动画预览时，区别其它动画的。
- finishedListener 可以用来监听动画的结束。

关于动画规格 AnimationSpec, 此处不展开，后面会详细讲解。

再延伸一点，看看该方法的实现，实际上是调用了 `animateValueAsState` 方法。事实上前面展示的 `animate*AsState` 的系列方法都是调用的 `animateValueAsState`。

看看源码：

```
@Composable
fun <T, V : AnimationVector> animateValueAsState(
    targetValue: T,
    typeConverter: TwoWayConverter<T, V>,
    animationSpec: AnimationSpec<T> = remember { spring() },
    visibilityThreshold: T? = null,
    label: String = "ValueAnimation",
    finishedListener: ((T) -> Unit)? = null
): State<T> {

    val toolingOverride = remember { mutableStateOf<State<T>?>(null) }
    val animatable = remember { Animatable(targetValue, typeConverter, visibilityThreshold, label) }
    val listener by rememberUpdatedState(finishedListener)
    val animSpec: AnimationSpec<T> by rememberUpdatedState(
        animationSpec.run {
            if (visibilityThreshold != null && this is SpringSpec &&
                this.visibilityThreshold != visibilityThreshold
            ) {
                spring(dampingRatio, stiffness, visibilityThreshold)
            } else {
                this
            }
        }
    )
    val channel = remember { Channel<T>(Channel.CONFLATED) }
    SideEffect {
        channel.trySend(targetValue)
    }
    LaunchedEffect(channel) {
        for (target in channel) {
            // This additional poll is needed because when the channel suspends on receive and
            // two values are produced before consumers' dispatcher resumes, only the first value
            // will be received.
            // It may not be an issue elsewhere, but in animation we want to avoid being one
            // frame late.
            val newTarget = channel.tryReceive().getOrNull() ?: target
            launch {
                if (newTarget != animatable.targetValue) {
                    animatable.animateTo(newTarget, animSpec)
                    listener?.invoke(animatable.value)
                }
            }
        }
    }
    return toolingOverride.value ?: animatable.asState()
}
```

稍微看下源码，大致能发现，实际上是启动了一个协程，然后协程内部不断调用了 `animatable.animateTo()` 这样一个方法。下一节讲 `animatable`, 收回来，这里我想要表达的意思是，使用接受通用类型的 `animateValueAsState()` 可以轻松添加对其他数据类型的支持只需要自行实现一个 `TwoWayConverter`。具体如何实现，下文第四节会详细讲解。

### 1.2 Animatable
前面的 `animate*AsState` 只需要指定目标值，无需指定初始值，而 Animatable 则是一个能够指定初始值的更基础的 API。 `animate*AsState` 调用了 `AnimateValueAsState`, 而 `AnimateValueAsState` 内部使用 `Animatable` 定制完成。

对于 Animatable 而言，动画数值更新需要在协程中完成，也就是调用 animateTo 方法。此时我们需要确保 Animatable 的初始状态与 LaunchedEffect 代码块首次执行时状态保持一致。

接下来，我们使用 animatable 实现前面的例子。

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }

    val customSize = remember {
        Animatable(50.dp, Dp.VectorConverter)
    }
    
    LaunchedEffect(key1 = bigBox) {
        customSize.animateTo(if (bigBox) 200.dp else 50.dp)
    }

    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(customSize.value)
        .clickable {
            bigBox = !bigBox
        })
}
```

值得注意的是，当我们在 Composable 使用 Animatable 时，其必须包裹在 rememebr 中，如果你没有这么做，编译器会贴心的提示你添加 rememeber 。

同样，与 animate*AsState 一样， animateTo 方法接收 `AnimationSpec` 参数用来指定动画的规格。

```
suspend fun animateTo(
        targetValue: T,
        animationSpec: AnimationSpec<T> = defaultSpringSpec,
        initialVelocity: T = velocity,
        block: (Animatable<T, V>.() -> Unit)? = null
    )
```

再看看增加一个颜色变化的动画：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }

    val customSize = remember {
        androidx.compose.animation.core.Animatable(50.dp, Dp.VectorConverter)
    }

    val customColor = remember {
        androidx.compose.animation.Animatable(Color.Blue)
    }

    LaunchedEffect(key1 = bigBox) {
        customSize.animateTo(if (bigBox) 200.dp else 50.dp)
        customColor.animateTo(if (bigBox) Color.Red else Color.Blue)
    }

    Box(modifier = Modifier
        .background(customColor.value)
        .size(customSize.value)
        .clickable {
            bigBox = !bigBox
        })
}
```

我们看到，前面使用 Dp 类型时，我们使用的是 `androidx.compose.animation.core.Animatable.kt` 文件中的方法， 此时使用的是 `androidx.compose.animation.SingleValueAnimation.kt` 文件中的方法, 且没有传入 TwoWayConverter 参数。这里说明一下：对与 `Color` 和 `Float` 类型，Compose 已经进行了封装，不需要我们传入 TwoWayConverter 参数，对于其它的常用数据类型，Compose 也提供了对应的 TwoWayConverter 实现方法。比如 `Dp.VectorConverter`, 直接传入即可。

另外 Launched 会在 onAtive 时执行，此时要确保， animateTo 的 targetValue 与 Animatable 的默认值相同。否则在页面首次渲染时，便会发生动画，可能与预期结果不相符。

最后我们看一下执行效果:

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic2.gif)

可以看到，实际上 size 和 Color 并不是同时执行的，而是先执行 size 的动画, 后执行 Color 的动画。我们做如下修改，让两个动画并发执行。

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }

    val customSize = remember {
        androidx.compose.animation.core.Animatable(50.dp, Dp.VectorConverter)
    }

    val customColor = remember {
        androidx.compose.animation.Animatable(Color.Blue)
    }

    LaunchedEffect(key1 = bigBox) {
        launch{
            customSize.animateTo(if (bigBox) 200.dp else 50.dp)
        }
        launch {
            customColor.animateTo(if (bigBox) Color.Red else Color.Blue)
        }
    }

    Box(modifier = Modifier
        .background(customColor.value)
        .size(customSize.value)
        .clickable {
            bigBox = !bigBox
        })
}
```

此时的执行效果，如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic3.gif)

对比看还是能看到其中差距，如果感觉不明显，可以在开发者模式里面将动画执行时间调整为 5 倍，则能更清晰地观察动画执行过程。这样看起来，size 和 color 动画是在异步并发执行。

### 1.3 Transition 动画
Animate*AsState 和 Animatable 都是针对单个目标值的动画，而 Transition 可以面向多个目标值应用动画，并保持它们同步结束。这听起来是不是类似传统 View 中的 AnimationSet ？

#### 1.3.1 updateTransition
我们可以使用 updateTransition 创建一个 Transition 动画，还是先上代码，看看 updateTransition 的用法：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }

    val transition = updateTransition(targetState = bigBox, label = "")
    val customSize by transition.animateDp(label = "") {
        when(it) {
            true -> 200.dp
            false -> 50.dp
        }
    }

    val customColor by transition.animateColor(label = "") {
        when(it) {
            true -> Color.Red
            false -> Color.Blue
        }
    }

    Box(modifier = Modifier
        .background(customColor)
        .size(customSize)
        .clickable {
            bigBox = !bigBox
        })
}
```

Transition 也需要依赖状态执行，需要枚举出所有可能的状态。然后基于这个状态，通过 `updateTransition` 创建了一个 Transition 对象，然后使用 transiton 动画，依次创建出 Size 和 Color。效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic4.gif)

咋一看，怎么感觉跟前面实现的 Size 和 Color 的调整后的这个动画是一样的？这里可以这样理解，Transition 动画是其依赖的状态变化后，会同步改变由它创建出来的其它多个属性，能保证各个属性同时变化，动画真正的同时启动，并同时结束。

#### 1.3.2 createChildTransition
通过 createChildTransition 可以将一种类型的 Transition 转换为其它的 Transition。
举例：

```
var bigBox by remember {
    mutableStateOf(false)
}

val transition = updateTransition(targetState = bigBox, label = "")

val sizeTransition = transition.createChildTransition(label = "") {
    when(it) {
        true -> 200.dp
        false -> 50.dp
    }
}


val colorTransition = transition.createChildTransition(label = "") {
    when(it) {
        true -> Color.Red
        false -> Color.Blue
    }
}
```

将一个 Transition 类型转换为了 Transition 和 Transition, 实际使用中，子动画的动画数值来自于父动画，某种程度上说，createChildTransition 更像是一种 map 操作。

#### 1.3.3 封装并复用 Transition 动画
使用 updateTransition 方法操作动画，没有问题，现在假设某个动画效果很复杂，我们不希望每次用的时候都去重新实现一遍，我们希望将上述动画效果封装起来，并可以复用。如何做呢？还是以上面的动画效果为例.
首先把动画涉及到的属性做一个封装：

```
class TransitionData(
    size: State<Dp>,
    color: State<Color>
) {
    val size by size
    val color by color
}
```

然后定义动画，并返回对应的值：

```
@Composable
fun ChangeBoxSizeAndColor(bigBox: Boolean): TransitionData {
    val transition = updateTransition(targetState = bigBox, label = "")
    val size = transition.animateDp(label = "") {
        when(it) {
            true -> 200.dp
            false -> 50.dp
        }
    }
    val color = transition.animateColor(label = "") {
        when(it) {
            true -> Color.Red
            false -> Color.Blue
        }
    }
    return remember (transition) {
        TransitionData(size, color)
    }
}
```

最后使用封装的动画：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    val sizeAndColor = ChangeBoxSizeAndColor(bigBox)
    Box(modifier = Modifier
        .background(sizeAndColor.color)
        .size(sizeAndColor.size)
        .clickable {
            bigBox = !bigBox
        })
}
```

执行结果和前面一致。这里就不在贴图了。

### 1.4 remeberInfiniteTransition —— 无限循环的 transition 动画
顾名思义，remeberInfiniteTransition 就是一个无限循环的 transition 动画。一旦动画开始便会无限循环下去，直到 Composable 进入 onDispose。
看下用法：

```
@Composable
fun Demo() {
    val infiniteTransition = rememberInfiniteTransition(label = "")
    val color by infiniteTransition.animateColor(
        initialValue = Color.Blue,
        targetValue = Color.Red,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = ""
    )
    val size by infiniteTransition.animateValue(
        initialValue = 50.dp,
        targetValue = 200.dp,
        typeConverter = Dp.VectorConverter,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = ""
    )

    Box(modifier = Modifier
        .background(color)
        .size(size)
    )
}
```

效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic5.gif)

首先使用 `remeberInfiniteTransition` 创建一个 InfiniteTransition 对象，然后通过该对象，创建出具体的要进行动画的属性，这里使用到了 animationSpec 动画规格，关于动画规格，不着急，下文马上讲解。需要注意到的是，`infiniteTransition.animateColor` 和 `infiniteTransition.animateFloat` 方法是不需要传入 typeConverter 参数的，其它类型，我们需要实现 TwoWayConverter。

### 1.5 小结
关于 Compose 低级别的动画 API ，我们介绍差不多了，主要是 `animtion*AsState`、`Animatable` 和 `Transition`，比如上面的例子中，我们用三种动画都实现了相同的效果。那这三者我们到底应该怎么理解？

>首先 `animtion*AsState` 底层实现实际上就是使用的 `Animatable`，我们可以把 `animtion*AsState` 理解为 `Animatable` 的一种更简便更直接的用法。这两个实际上可以归为同一类。
而 Transition 的核心思想与 Animatable 不一样。**Animatable 的核心思想是面向值的**，在多个动画，多个状态的情况下，存在不方便管理的问题。比如针对 size 和 color，我们需要创建出两个 Animatable 对象，并且需要启动两个协程，如果有更多的还要同时执行更多的动画，则会更复杂。**Transition 的核心思想是面向状态的，如果多个动画依赖一个共同的状态，则可以做到统一管理**。updateTransition 只会创建一次协程，根据一种状态的变化，控制不同的动画效果。很明显，transition 在代码结构上以及逻辑上更清晰。

另外，Transition 还支持一个非常牛叉的功能：支持 Compose 动画预览调节!!!

## 二、Android Studio 对 Compose 动画调试的支持
为了方便后面在体验不同动画规格时，感受效果差异，在讲解动画规格之前，这里插入一节，将一下 Android Studio 对与 Compose 动画的预览调试，这个是真正的生产力。我们不需要在编译 apk 在真机或者模拟器上运行，就可以预览动画效果，这个是不是大大提高了效率。怎么用？—— `@Preview` 注解。
首先，不论是 Animatable 动画还是 Transition 动画， Android Studio 都支持预览功能。
还是上面的例子：

```
@Preview
@Composable
fun Demo1() {
    var bigBox by remember {
        mutableStateOf(false)
    }

    val customSize = remember {
        androidx.compose.animation.core.Animatable(50.dp, Dp.VectorConverter, label = "demo1Size")
    }

    val customColor = remember {
        androidx.compose.animation.Animatable(Color.Blue)
    }

    LaunchedEffect(key1 = bigBox) {
        launch {
            customSize.animateTo(if (bigBox) 200.dp else 50.dp)
        }
        launch {
            customColor.animateTo(if (bigBox) Color.Red else Color.Blue)
        }
    }

    Box(modifier = Modifier
        .background(customColor.value)
        .size(customSize.value)
        .clickable {
            bigBox = !bigBox
        })
}

@Preview
@Composable
fun Demo2() {
    var bigBox by remember {
        mutableStateOf(false)
    }

    val transition = updateTransition(targetState = bigBox, label = "demo2SizeAndColor")
    val customSize by transition.animateDp(label = "demo2Size") {
        when(it) {
            true -> 200.dp
            false -> 50.dp
        }
    }

    val customColor by transition.animateColor(label = "demo2Color") {
        when(it) {
            true -> Color.Red
            false -> Color.Blue
        }
    }

    Box(modifier = Modifier
        .background(customColor)
        .size(customSize)
        .clickable {
            bigBox = !bigBox
        })
}
```

1. 给需要预览调节的 Composable 加上 @Preview 注解。这里我们还对动画函数中的 label 参数重新赋值了。这个 label 参数就是方便预览调节的。
2. 进入 Android Studio 预览模式

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic6.png)

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic7.png)

然后将鼠标移动到对应的 Composable 名称上，可以看到有多种模式：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic8.png)

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic9.png)

一般会有如下预览模式:
`start UI check mode` 用来检查 UI 状态，比如横竖屏，暗黑模式，亮色模式，rtl 布局，不同尺寸的设备，不同主题等 UI 适配问题，非常方便。
`run preview` 是在真机或者模拟器中运行当前的 Composable，进行预览，切合实际场景。
`start interact mode` 启动交互模式，顾名思义，比如点击、长按，拖拽等等事件交互进行预览，当然动画也是一种交互形式，可以在这个模式下进行动画预览。
`start animation preview` 启动动画预览模式，这个就是上一节提到的，Transition 动画特有的预览调节模式。

我们可以看到只有 Demo2 有 `start animation preview` 选项。启动该模式：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic10.png)

①区域，可以播放动画，是否循环，播放速度，跳转到动画起始位置或者结束位置。
②区域，可以设置动画依赖的状态变化。
③区域，可以展开或者收起具体的动画，展开后，④区域会显示当前具体每个动画执行过程，前面个动画设置 label, 就是用作在此做区别的。并且有一个时间轴，可以自行调节。

虽然 animate*AsState 也支持添加 label，但是该类型动画只支持预览，不支持调节。但即便如此，是不是比传统 View 方便了一万倍。更多预览调试功能大家可以自行探索。

## 三、AnimationSpec 动画规格
终于讲到动画规格了。动画规格实际上就是控制动画如何执行的。前面出现的代码中多次提到 animationSpec 参数，大多数 Compose 动画 API 都支持设置 animationSpec 参数定义动画效果。前面我们没有传入这个参数，是因为使用到的 API 该参数都有一个默认值。
比如：

```
@Composable
fun animateDpAsState(
    targetValue: Dp,
    animationSpec: AnimationSpec<Dp> = dpDefaultSpring,
    label: String = "DpAnimation",
    finishedListener: ((Dp) -> Unit)? = null
): State<Dp> {
    return animateValueAsState(
        targetValue,
        Dp.VectorConverter,
        animationSpec,
        label = label,
        finishedListener = finishedListener
    )
}
```

animateDpAsState 方法对于 animationSpec 参数赋了默认值 dpDefaultSpring。我们先看看这个 AnimationSpec 定义：

```
interface AnimationSpec<T> {
    fun <V : AnimationVector> vectorize(
        converter: TwoWayConverter<T, V>
    ): VectorizedAnimationSpec<V>
}
```

AnimationSpec 是一个接口，只有一个方法，泛型 T 是当前动画的数值类型，vectorize 方法用来创建一个 VectorizedAnimationSpec，这是一个矢量动画的配置。AnimationVector 其实就是一个函数，用来参与计算动画矢量，TwoWayConverter 用来将 T 类型转换成参与动画计算的矢量数据。
AnimationSpec 的实现类如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic11_0.png)

下面按照类别介绍：

### 3.1 SpringSpec 弹跳动画
Spring 弹性动画。可以使用 spring() 方法进行创建。

```
@Stable
fun <T> spring(
    dampingRatio: Float = Spring.DampingRatioNoBouncy,
    stiffness: Float = Spring.StiffnessMedium,
    visibilityThreshold: T? = null
): SpringSpec<T> =
    SpringSpec(dampingRatio, stiffness, visibilityThreshold)
```

spring 方法接收三个参数，都有默认值。

1. **dampingRatio：** 弹簧的阻尼比，阻尼比可以定义震动从一次弹跳到下一次弹跳所衰减的速度有多快。当阻尼比 < 1 时，阻尼比越小，弹簧越有弹性，默认值为 `Spring.DampingRatioNoBouncy = 1f`。
- 当 dampingRatio > 1 时会出现过阻尼现象，这会使弹簧快速地返回到静止状态。
- 当 dampingRatio = 1 时，没有弹性的阻尼比，会使得弹簧在最短的时间内返回到静止状态。
- 当 0 < dampingRatio < 1 时, 弹簧会围绕最终静止为止多次反复振动。

注意 dampingRatio 不能小于0。
Compose 为 spring 提供了一组常用的阻尼比常量。

```
const val DampingRatioHighBouncy = 0.2f

const val DampingRatioMediumBouncy = 0.5f

const val DampingRatioLowBouncy = 0.75f

const val DampingRatioNoBouncy = 1f
```

2. **stiffness：** 弹簧的刚度，刚度值越大，弹簧到静止的速度就越快。默认值为 `Spring.StiffnessMedium = 1500f`。
注意 stiffness 必须大于 0 。
Compose 为 spring 提供了一组常量值：

```
const val StiffnessHigh = 10_000f

const val StiffnessMedium = 1500f

const val StiffnessMediumLow = 400f

const val StiffnessLow = 200f

const val StiffnessVeryLow = 50f
```

3. **visibilityThreshold:** 可见性阈值。这个参数是一个泛型，次泛型与 targetValue 的类型保持一致，又开发者指定一个阈值，当动画达到这个阈值时，动画会立即停止。默认值为 null。
示例：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    val customSize by animateDpAsState(
        targetValue = if (bigBox) 200.dp else 50.dp,
        animationSpec = spring(dampingRatio = -0.2f),
        label = ""
    )
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(customSize)
        .clickable {
            bigBox = !bigBox
        })
}
```

这个 Gif 录制效果着实不太好，大家可以试一下，实际效果很明显，像弹框一样反复振动直到静止。

### 3.2 TweenSpec 补间动画
TweenSpec 是 DurationBasedAnimationSpec 的子类。TweenSpec 的动画必须在规定的时间内完成，它的动画效果是基于时间参数计算的，可以使用 Easing 来指定不同的时间曲线动画效果。可以使用 tween() 方法进行创建。

```
@Stable
fun <T> tween(
    durationMillis: Int = DefaultDurationMillis,
    delayMillis: Int = 0,
    easing: Easing = FastOutSlowInEasing
): TweenSpec<T> = TweenSpec(durationMillis, delayMillis, easing)
```

tween 方法接收三个参数：

1. **durationMillis:** 动画的持续时间，默认值 300ms。
2. **delayMillis:** 动画延迟时间，默认值 0, 即立即执行。
3. **easing:** 动画曲线变化，默认值为 FastOutSlowInEasing。

**Easing**
下面介绍一下 Easing,

```
@Stable
fun interface Easing {
    fun transform(fraction: Float): Float
}
```

Easing 是一个接口，只有一个方法， transform 用来计算变化曲线，允许过度元素加速或者减速，而不是以恒定的速度变化。参数 fraction 是一个 [0.0f, 1.0f] 区间的值。其中 0.0f 表示 开始，1.0f 表示结束。
Compose 给出了 Easing 的两个实现类：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic11.png)

**PathEasing**

```
class PathEasing(path: Path) : Easing
```

PathEasing 需要创建一个 Path 对象传入。关于 Path 的使用，和 View 中的 Path API 有一些类似，具体可以查看源码，需要注意的是这个 Path 必须从 (0,0) 开始，到 (1, 1) 结束，并且 x 方向上不得有间隙，也不得自行循环，避免有两个点共享相同的 x 坐标。

**CubicBezierEasing**

```
class CubicBezierEasing(
    private val a: Float,
    private val b: Float,
    private val c: Float,
    private val d: Float
) : Easing
```

实际开发中，使用更多的是三阶贝塞尔曲线。a、b, 是第一个控制的 x 坐标和 y 坐标，c、d 是第二个控制点的 x 坐标和 y 坐标。
关于三阶贝塞尔曲线的知识，本文就展开了，如不清楚的可以查阅相关资料。Compose 提供了一些常用的实现。

```
val FastOutSlowInEasing: Easing = CubicBezierEasing(0.4f, 0.0f, 0.2f, 1.0f)

val LinearOutSlowInEasing: Easing = CubicBezierEasing(0.0f, 0.0f, 0.2f, 1.0f)

val FastOutLinearInEasing: Easing = CubicBezierEasing(0.4f, 0.0f, 1.0f, 1.0f)
```

如需自定义控制点坐标，可以到一些在线网站上预览曲线变化效果。比如：

**LinearEasing**
除此之外，还有一个线性变化曲线。

```
val LinearEasing: Easing = Easing { fraction -> fraction }
```

**自定义 Easing**
自定义 Easing 实际上就是自己显示 Easing 接口。
例如：

```
val CustomEasing = Easing { fraction -> fraction * fraction }
```

### 3.3 KeyframesSpec 关键帧动画
KeyframesSpec 也是 DurationBasedAnimationSpec，基于时间的动画规格，在不同的时间戳定义值，更精细地来实现关键帧的动画。可以使用 keyframes() 方法来创建 KeyframesSpec。

```
@Stable
fun <T> keyframes(
    init: KeyframesSpec.KeyframesSpecConfig<T>.() -> Unit
): KeyframesSpec<T> {
    return KeyframesSpec(KeyframesSpec.KeyframesSpecConfig<T>().apply(init))
}
```

keyframes 方法只有一个参数。直接看用法：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    val customSize by animateDpAsState(
        targetValue = if (bigBox) 200.dp else 50.dp,
        animationSpec = keyframes {
            durationMillis = 10000
            50.dp at 0 with LinearEasing
            100.dp at 1000 with FastOutLinearInEasing
            150.dp at 9000 with LinearEasing
            200.dp at 10000
        },
        label = ""
    )
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(customSize)
        .clickable {
            bigBox = !bigBox
        })
}
```

解释：指定在什么时间，值应该是多少，是以怎样的曲线变化到该值。

注意，这个示例有个奇怪的点，由小变到大是符合逻辑的，而由大变大小，这个效果会显得很奇怪。

### 3.4 SnapSpec 跳切动画
SnapSpec 表示跳切动画，它立即将动画值捕捉到最终值。它的 targetValue 发生变化时，当前值会立即更新为 targetValue, 没有中间过渡，动画会瞬间完成，常用于跳过过场动画的场景。使用 snap() 方法创建。

```
@Stable
fun <T> snap(delayMillis: Int = 0) = SnapSpec<T>(delayMillis)
```

接收一个参数，表示延迟多久后执行。

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    val customSize by animateDpAsState(
        targetValue = if (bigBox) 200.dp else 50.dp,
        animationSpec = snap(),
        label = ""
    )
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(customSize)
        .clickable {
            bigBox = !bigBox
        })
}
```

### 3.5 RepeatableSpec 循环动画
使用 repeatable() 方法可以创建一个 RepeatableSpec 示例，前面介绍的动画规格都是单次执行的动画，而 RepeatableSpec 是一个可循播放的动画。

```
@Stable
fun <T> repeatable(
    iterations: Int,
    animation: DurationBasedAnimationSpec<T>,
    repeatMode: RepeatMode = RepeatMode.Restart,
    initialStartOffset: StartOffset = StartOffset(0)
): RepeatableSpec<T> =
    RepeatableSpec(iterations, animation, repeatMode, initialStartOffset)
```

repeatable 有四个参数：

1. **iterations：** 循环次数，理论上应该大于 1 ，等于 1 表示不循环。那也就没有必要使用 RepeatableSpec 了。
2. **animation：** 该参数是一个 DurationBasedAnimationSpec 类型。可以使用
`TweenSpec`、`KeyframesSpec`、`SnapSpec`。SpringSpec 不支持循环播放，这个可以理解，循环的弹性，违背物理定律。
3. **repeatMode：** 重复模式，枚举类型。

```
enum class RepeatMode {
    Restart,

    Reverse
}
```

示例：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    val customSize by animateDpAsState(
        targetValue = if (bigBox) 200.dp else 50.dp,
        animationSpec = repeatable(
            iterations = 10,
            animation = tween()
        ),
        label = ""
    )
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(customSize)
        .clickable {
            bigBox = !bigBox
        })
}
```

4. **initialStartOffset:** 动画开始的偏移。可用于延迟动画的开始或将动画快进到给定的播放时间。此起始偏移量不会重复，而动画中的延迟（如果有）将重复。 默认情况下，偏移量为 0。

### 3.6 InfiniteRepeatableSpec 无限循环动画
InfiniteRepeatableSpec 表示无限循环的动画，使用 infiniteRepeatable() 方法创建。

```
@Stable
fun <T> infiniteRepeatable(
    animation: DurationBasedAnimationSpec<T>,
    repeatMode: RepeatMode = RepeatMode.Restart,
    initialStartOffset: StartOffset = StartOffset(0)
): InfiniteRepeatableSpec<T> =
    InfiniteRepeatableSpec(animation, repeatMode, initialStartOffset)
```

与 repeatable() 方法相比少了一个参数 iterations。无限循环动画自然是不需要指定重复次数的，其余参数一样。
示例：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    val customSize by animateDpAsState(
        targetValue = if (bigBox) 200.dp else 50.dp,
        animationSpec = infiniteRepeatable(
            animation = tween()
        ),
        label = ""
    )
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(customSize)
        .clickable {
            bigBox = !bigBox
        })
}
```

这里注意和前面讲到的 `remeberInfiniteTransition` 用法做一个区分。
remeberInfiniteTransition 是一种动画类型，需要结合 infiniteRepeatable 一起使用，是在 transition 动画中使用，而 infiniteRepeatableSpec 是一种动画规格，可以使用在任何需要 animationSpec 参数的方法中。

### 3.7 FloatAnimationSpec
`FloatAnimationSpec` 是一个接口，有两个实现类，`FloatTweenSpec` 仅针对 Float 类型做 TweenSpec 动画，FloatSpringSpec 仅针对 Float 类型做 SpringSpec 动画。官方没有提供可以直接进行使用的方法，因为 tween() 和 spring() 支持全量数据类型，FloatAnimationSpec 是底层做更精细的计算的时候才会去使用。

### 3.8 KeyframesWithSplineSpec
KeyframesWithSplineSpec 是基于三次埃尔米特样条曲线的变化，截止目前，该类是一个实验性质的 API， 目前官方文档没有说明其使用方法。等待后续稳定可用。

关于动画规格的介绍这些了。

## 四、TwoWayConverter
### 4.1 TwoWayConterver 是什么
TwoWayConterver 是一个接口：

```
interface TwoWayConverter<T, V : AnimationVector> {
    val convertToVector: (T) -> V

    val convertFromVector: (V) -> T
}
```

它可以需要实现将任意 T 类型的数值转换成标准的 AnimationVector 类型。以及将标准的 AnimationVector 类型转换为任意的 T 类型数值。

这个 AnimationVerction 有如下子类：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic12.png)

分别表示 1 维到 4 维的的矢量值。

我们看看几个 Compose 实现的方式。

```
private val DpToVector: TwoWayConverter<Dp, AnimationVector1D> = TwoWayConverter(
    convertToVector = { AnimationVector1D(it.value) },
    convertFromVector = { Dp(it.value) }
)
```

```
private val SizeToVector: TwoWayConverter<Size, AnimationVector2D> =
    TwoWayConverter(
        convertToVector = { AnimationVector2D(it.width, it.height) },
        convertFromVector = { Size(it.v1, it.v2) }
    )
```

```
private val OffsetToVector: TwoWayConverter<Offset, AnimationVector2D> =
    TwoWayConverter(
        convertToVector = { AnimationVector2D(it.x, it.y) },
        convertFromVector = { Offset(it.v1, it.v2) }
    )
```

可以看到，常用类型都已经提供了 TwoWayConverter 的拓展实现。可以在这些类型的半生对象中找到，并可以直接使用。

### 4.2 自定义 TwoWayConterver
对于没有提供默认支持的数据类型，可以自定义 TwoWayConterver。
示例：

```
data class HealthData(val height: Float, val weight: Float)

@Composable
fun MyAnimation(targetHealthData: HealthData) {
    val healthDataToVector: TwoWayConverter<HealthData, AnimationVector2D> = TwoWayConverter(
        convertToVector = { AnimationVector2D(it.height, it.weight) },
        convertFromVector = { HealthData(it.v1, it.v2) }
    )

    val animationHeath by animateValueAsState(
        targetValue = targetHealthData,
        typeConverter = healthDataToVector, 
        label = ""
    )
}
```

我们定义了一个健康数据类，包含身高和体重。然后实现一个 TwoWayConterver。并基于这个 TwoWayConterver 实现了一个自定义动画。

## 五、高级动画 API
所谓高级别动画 API ，是指这些 API 是基于前面讲到的低级别动画 API 进行封装的，使用起来更方便。
很多 Compose 教程都是先介绍高级别动画 API ，再介绍低级别动画 API，因为高级别的 API 使用起来更简单，服务于常见的业务，开箱即用。我先介绍了低级别动画 API ,是因为我觉得这样更好理解。明白了背后的原理，再去看上层实现，才能心中的脉络框架。下面逐一介绍高级别的动画 API。

### 5.1、AnimatedVisibility
#### 5.1.1 基本使用
AnimatedVisibility 是一个用于可组合项出现/消失的过渡动画效果。借助 AnimatedVisibility 可以轻松实现隐藏和显示可组合项。
看一下效果：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic13.gif)

使用方法如下：

```
var visible by remember {
    mutableStateOf(true)
}
// Animated visibility will eventually remove the item from the composition once the animation has finished.
AnimatedVisibility(visible) {
    // your composable here
    // ...
}
```

AnimatedVisibility 本身就是一个 Composable。定义如下：

```
@Composable
fun AnimatedVisibility(
    visible: Boolean,
    modifier: Modifier = Modifier,
    enter: EnterTransition = fadeIn() + expandIn(),
    exit: ExitTransition = shrinkOut() + fadeOut(),
    label: String = "AnimatedVisibility",
    content: @Composable() AnimatedVisibilityScope.() -> Unit
) {
    val transition = updateTransition(visible, label)
    AnimatedVisibilityImpl(transition, { it }, modifier, enter, exit, content = content)
}
```

可以看到，AnimatedVisibility 是基于 Transition 动画实现的。关键参数如下：

- **visible：** 表示可组合项出现或者消失。
- **content：** 表示要添加出现/消失动画的可组合项内容。
- **enter：** 表示 content 出现的动画。EnterTransition 类型。
- **exit：** 表示 content 消失的动画。ExitTransition 类型。

对于动画，可以使用 `+` 运算符，组合多个 EnterTransition 或者 ExitTransition 对象。默认情况下内容以淡入和扩大的方式出现，以淡出和缩小的方式消失。
Compose 提供了多种 EnterTransition 或者 ExitTransition 的实例。
`fadeIn`、`fadeOut`、`slideIn`、`slideOut`、`slideInHorizontally`、`slideOutHorizontally`、`slideInVertically`、`slideOutVertically`、`scaleIn`、`scaleOut`、`expandIn`、`shrinkOut`、`expandHorizontally`、`shrinkHorizontally`、`expandVertically`、`shrinkVertically`。
具体效果大家可以动手试一下，也可以到官网查看：[animatedvisibility](https://developer.android.com/develop/ui/compose/animation/composables-modifiers?hl=zh-cn#animatedvisibility)

示例：

```
@Composable
fun Demo() {
    var visible by remember {
        mutableStateOf(false)
    }

    Column {
        AnimatedVisibility(visible) {
            Box(modifier = Modifier.background(Color.LightGray).size(200.dp))
        }
        Button(onClick = {
            visible = !visible
        }) {
            Text(text = "click me")
        }
    }
}
```

效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic14.gif)

#### 5.1.2 MutableTransitionState 监听动画执行状态
AnimatedVisibility 还提供了另一个重载方法。它接受一个 `MutableTransitionState` 类型的参数。

```
@Composable
fun AnimatedVisibility(
    visibleState: MutableTransitionState<Boolean>,
    modifier: Modifier = Modifier,
    enter: EnterTransition = fadeIn() + expandIn(),
    exit: ExitTransition = fadeOut() + shrinkOut(),
    label: String = "AnimatedVisibility",
    content: @Composable() AnimatedVisibilityScope.() -> Unit
) {
    val transition = updateTransition(visibleState, label)
    AnimatedVisibilityImpl(transition, { it }, modifier, enter, exit, content = content)
}
```

它与上一小节的那个方法，只有第一个参数不一样，它支持监听动画状态。看一下 `MutableTransitionState` 的定义：

```
class MutableTransitionState<S>(initialState: S) : TransitionState<S>() {

    override var currentState: S by mutableStateOf(initialState)
        internal set

    override var targetState: S by mutableStateOf(initialState)

    val isIdle: Boolean
        get() = (currentState == targetState) && !isRunning

    override fun transitionConfigured(transition: Transition<S>) {
    }
}
```

MutableTransitionState 有两个关键成员：`currentState` 和 `targetState`，表示当前状态和目标状态。两个状态的不同驱动了动画的执行。用法如下：

```
@Composable
fun Demo() {
    val visible = remember {
        MutableTransitionState<Boolean>(false)
    }

    Column {
        AnimatedVisibility(visible) {
            Box(modifier = Modifier.background(Color.LightGray).size(200.dp))
        }
        Button(onClick = {
            visible.targetState = !visible.currentState
        }) {
            Text(text = "click me")
        }
    }
}
```

当需要执行动画时，只需要改变 MutableTransitionState 对象的 targetState，让它与 currentState 不同。效果与上一小节完全一致，这里就不贴图了。
另外 MutableTransitionState 还可以方便实现 AnimatedVisibility 首次添加到组合树中，就立即触发动画。只需在初始化 MutableTransitionState 对象是，让 targetState 和 currentState 不同即可。可以用此来实现一些开屏动画的效果。

```
val visible = remember {
    MutableTransitionState<Boolean>(false).apply {
        targetState = true
    }
}
```

此外，MutableTransitionState 的意义还在于可以通过 currentState 和 isIdle 的值，获取动画的执行状态。

```
@Composable
fun Demo() {
    val visible = remember {
        MutableTransitionState<Boolean>(false)
    }

    Column {
        AnimatedVisibility(visible) {
            Box(
                modifier = Modifier
                    .background(Color.LightGray)
                    .size(200.dp)
            )
        }
        Button(onClick = {
            visible.targetState = !visible.currentState
        }) {
            Text(text = "click me")
        }
        Text(
            text = when {
                visible.isIdle && visible.currentState -> "Visible"
                !visible.isIdle && visible.currentState -> "Disappearing"
                visible.isIdle && !visible.currentState -> "Invisible"
                else -> "Appearing"
            }
        )
    }
}
```

效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic15.gif)

#### 5.1.3 为子可组合项添加进入和退出的动画
AnimatedVisibility 中直接或者间接的子可组合项，可以使用 `Modifier.animateEnterExit` 修饰符单独设置进入和退出的过渡动画。这样子项的动画就是 AnimatedVisibility 中设置的动画与子项自己设置的动画结合在一起构成的。

```
@OptIn(ExperimentalAnimationApi::class)
@Composable
fun Demo() {
    var visible by remember {
        mutableStateOf(false)
    }
    Column {
        AnimatedVisibility(
            visible = visible,
        ) {
            Box(
                modifier = Modifier
                    .background(Color.LightGray)
                    .size(200.dp)
                    .animateEnterExit(
                        enter = slideInHorizontally(),
                        exit = slideOutHorizontally(),
                        label = ""
                    )
            )
        }
        Button(onClick = {
            visible = !visible
        }) {
            Text(text = "click me")
        }
    }
}
```

如果希望每个子项完全自己定义不同的动画效果，则可以将 AnimatedVisibility 中的动画设置为 `EnterTransition.None` 和 `ExitTransition.None`。

#### 5.1.4 自定义 AnimatedVisibility 动画
除了使用 Compose 提供的 `EnterTransition` 和 `ExitTransition` 动画以外，AnimatedVisibility 还支持自定义动画效果。通过在 AnimatedVisibility 的 AnimatedVisibilityScope 中的 transition 访问底层的 Transition 实例。添加到 transition 的动画会和 AnimatedVisibility 中设置的动画同时运行，AnimatedVisibility 会等到 Transition 中的所有动画都完成后，再移除其内容。对于独立于 Transition 创建的动画（比如使用 `animate*AsState` 创建的动画）, AnimatedVisibility 将无法解释这些动画。因此可能会在动画完成之前移除内容可组合项。

```
@OptIn(ExperimentalAnimationApi::class)
@Composable
fun Demo() {
    var visible by remember {
        mutableStateOf(false)
    }
    Column {
        AnimatedVisibility(
            visible = visible,
        ) {
            val bgColor by transition.animateColor(label = "") { state ->
                if (state == EnterExitState.Visible) Color.Red else Color.Blue
            }
            Box(
                modifier = Modifier
                    .background(bgColor)
                    .size(200.dp)
            )
        }
        Button(onClick = {
            visible = !visible
        }) {
            Text(text = "click me")
        }
    }
}
```

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic16.gif)

直白点理解就是 AnimatedVisibilityScope 里面提供了一个 trasition 对象，然后通过 transition 对象做动画处理。
这也是我为什么先讲解低级别动画 API 的原因，不然在这里自定义动画，可能有些术语就会造成困扰。
我们是否也可以自己使用前面讲解的 `updateTransition()` 方法创建一个 trasition 对象，然后使用这个 trasition 做动画处理呢？可以是可以，但是与使用 `animate*AsState` 一样，AnimatedVisibility 将无法解释这些动画。可能会在动画完成之前移除内容可组合项。
如下代码做了一个对比：

```
@Preview
@OptIn(ExperimentalAnimationApi::class)
@Composable
fun Demo1() {
    var visible by remember {
        mutableStateOf(false)
    }
    Column {
        AnimatedVisibility(
            visible = visible,
        ) {
            val bgColor by transition.animateColor(label = "bgColor",
                transitionSpec = {
                    tween(durationMillis = 3000, delayMillis = 0, easing = LinearEasing)
                }
            ) { state ->
                if (state == EnterExitState.Visible) Color.Red else Color.Blue
            }

            Box(
                modifier = Modifier
                    .background(bgColor)
                    .size(200.dp)
            )
        }
        Button(onClick = {
            visible = !visible
        }) {
            Text(text = "click me")
        }
    }
}

@Preview
@Composable
fun Demo2() {
    var visible by remember {
        mutableStateOf(false)
    }
    Column {
        AnimatedVisibility(
            visible = visible,
        ) {
            val myTransition = updateTransition(targetState = visible, label = "myColor")
            val myColor by myTransition.animateColor(
                label = "",
                transitionSpec = {
                    tween(durationMillis = 3000, delayMillis = 0, easing = LinearEasing)
                }
            ) {
                if (it) Color.Red else Color.Blue
            }

            Box(
                modifier = Modifier
                    .background(myColor)
                    .size(200.dp)
            )
        }
        Button(onClick = {
            visible = !visible
        }) {
            Text(text = "click me")
        }
    }
}
```

此处特意将动画持续时间设置为了 3000ms， 最终的效果就是，Demo1 中，在 enter 时，我们能看到颜色渐变动画的整个过程。而 Demo2 就感觉不明显。
使用动画调节器可以更清晰看到区别：
Demo1:

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic17.png)

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic18.png)

当我们使用 AnimatedVisibilityScope 里面提供的 transition 对象做动画，所有的动画都包含在 AnimatedVisility 这个动画之内。以最长的时间为动画执行时间。
Demo2:

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic19.png)

当我们通过 updateTransition() 创建 transition 对象做动画，这个动画和 AnimatedVisility 动画是独立的。

### 5.2 AnimatedContent
AnimatedContent 可组合项会在内容根据目标状态发生变化时，为内容添加动画效果。

#### 5.2.1 基本用法
用法：

```
@Composable
fun Demo() {
    var typeState by remember {
        mutableStateOf(true)
    }
    Column {
        Button(onClick = {
            typeState = !typeState
        }) {
            Text(text = "Change Type")
        }
        AnimatedContent(targetState = typeState, label = "") {
            if (it) {
                Text(
                    text = "Hello, Animation",
                    modifier = Modifier
                        .background(Color.Yellow)
                        .wrapContentSize()
                )
            } else {
                Icon(imageVector = Icons.Filled.Face, contentDescription = "")
            }
        }
    }
}
```

为了清楚看到效果，我在开发者模式，把动画持续时间改为 5 倍了。

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic20.gif)

看一下定义：

```
@Composable
fun <S> AnimatedContent(
    targetState: S,
    modifier: Modifier = Modifier,
    transitionSpec: AnimatedContentTransitionScope<S>.() -> ContentTransform = {
        (fadeIn(animationSpec = tween(220, delayMillis = 90)) +
            scaleIn(initialScale = 0.92f, animationSpec = tween(220, delayMillis = 90)))
            .togetherWith(fadeOut(animationSpec = tween(90)))
    },
    contentAlignment: Alignment = Alignment.TopStart,
    label: String = "AnimatedContent",
    contentKey: (targetState: S) -> Any? = { it },
    content: @Composable() AnimatedContentScope.(targetState: S) -> Unit
) {
    val transition = updateTransition(targetState = targetState, label = label)
    transition.AnimatedContent(
        modifier,
        transitionSpec,
        contentAlignment,
        contentKey,
        content = content
    )
}
```

简单理解 `AnimatedContent` 这个可组合函数，AnimatedContent 内部维护着 targetState 到 content 的映射表，当 targetState 发生变化了， AnimatedContentScope 中的可组合函数 content 就会发生重组，在 content 重组时附加的动画效果就会执行。

默认情况下，动画效果是初始内容淡出，目标内容淡入。可以从 AnimatedContent 的声明中看到。自定义动画效果，即给 transitionSpec 参数指定一个 `ContentTransform` 对象。ContentTransform 也是由 EnterTransition 和 ExitTranstion 组合的，可以使用中缀运算符 with 将 EnterTransition 和 ExitTransition 组合起来以常见 ContentTransform。

```
infix fun EnterTransition.togetherWith(exit: ExitTransition) = ContentTransform(this, exit)
```

我当前是 Compose 1.6.5 版本，比较新，显示 with 已经过时了。现在使用 `togetherWith`，用法一样。

```
infix fun EnterTransition.togetherWith(exit: ExitTransition) = ContentTransform(this, exit)
```

下面看下例子：

```
@Composable
fun Demo() {
    var typeState by remember {
        mutableStateOf(true)
    }
    Column {
        Button(onClick = {
            typeState = !typeState
        }) {
            Text(text = "Change Type")
        }
        AnimatedContent(targetState = typeState, label = "", transitionSpec = {
            slideInVertically { fullHeight -> fullHeight }.togetherWith(slideOutVertically { fullHeight -> -fullHeight })
        }) {
            if (it) {
                Text(
                    text = "Hello, Animation",
                    modifier = Modifier
                        .background(Color.Yellow)
                        .wrapContentSize()
                )
            } else {
                Icon(imageVector = Icons.Filled.Face, contentDescription = "")
            }
        }
    }
}
```

效果：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic21.gif)

另外，AnimatedContent 提供了 `slideIntoContainer` 和 `slideOutOfContainer`，可以作为 `slideInHorizontally/Vertically` 和 `slideOutHorizontally/Vertically` 的便捷替代方案。它们可以根据 另外，AnimatedContent 的初始内容的大小和目标内容的大小来计算滑动距离。

#### 5.2.2 SizeTransform 定义大小动画
AnimatedContent 中，还可以使用中缀函数 `using` 将 `SizeTransform` 应用于 ContentTransform 来定义大小动画。
看下 SizeTransform ：

```
fun SizeTransform(
    clip: Boolean = true,
    sizeAnimationSpec: (initialSize: IntSize, targetSize: IntSize) -> FiniteAnimationSpec<IntSize> =
        { _, _ ->
            spring(
                stiffness = Spring.StiffnessMediumLow,
                visibilityThreshold = IntSize.VisibilityThreshold
            )
        }
): SizeTransform = SizeTransformImpl(clip, sizeAnimationSpec)
```

`SizeTransform` 定义了大小应如何在初始内容与目标内容之间添加动画效果。在创建动画时，可以访问初始大小和目标大小。`SizeTransform` 还可控制在动画播放期间是否应将内容裁剪为组件大小。

示例：

```
@Composable
fun Demo() {
    var typeState by remember {
        mutableStateOf(true)
    }
    Column {
        Button(onClick = {
            typeState = !typeState
        }) {
            Text(text = "Change Type")
        }
        AnimatedContent(targetState = typeState, label = "", transitionSpec = {
            fadeIn(animationSpec = tween(150, 150))
                .togetherWith(fadeOut(animationSpec = tween(150)))
                .using(
                    SizeTransform(
                        clip = true,
                        sizeAnimationSpec = { initSize, targetSize ->
                            if (targetState) {
                                keyframes {
                                    // 展开时，先水平方向展开.
                                    IntSize(targetSize.width, initSize.height) at 150
                                    durationMillis = 300
                                }
                            } else {
                                keyframes {
                                    // 收起时，先垂直方向收起.
                                    IntSize(initSize.width, targetSize.height) at 150
                                    durationMillis = 300
                                }
                            }
                        }
                    )
                )
        }) {
            if (it) {
                Text(
                    text = "Hello, Animation",
                    modifier = Modifier
                        .background(Color.Yellow)
                        .wrapContentSize()
                )
            } else {
                Icon(imageVector = Icons.Filled.Face, contentDescription = "")
            }
        }
    }
}
```

#### 5.2.3 为子可组合项添加动画效果
与 AnimatedVisibility 一样， AnimatedContent 直接或间接的子可组合项都可以使用修饰符 `Modifier.animateEnterExit` 为自身添加动画，这里不再赘述。

#### 5.2.4 自定义 AnimatedContent 动画效果
与 AnimatedVisibility 一样， AnimatedContent 中的 AnimatedContentScope 内部提供了一个 `transition`, 可以用来自定义动画效果，这里不再赘述。

#### 5.3 Crossfade
Crossfade 可使用淡入淡出动画在两个布局之间添加动画效果。通过切换传递给 current 参数的值，可以使用淡入淡出动画来切换内容。可以理解为 AnimatedContent 的一种功能特性。如果只需要淡入淡出动画效果，则可以使用 Crossfade 来替代 AnimatedContent。
使用方式：

```
@Composable
fun Demo() {
    var typeState by remember {
        mutableStateOf(true)
    }
    Column {
        Button(onClick = {
            typeState = !typeState
        }) {
            Text(text = "Change Type")
        }
        Crossfade(targetState = typeState, label = "") {
            if (it) {
                Text(
                    text = "Hello, Animation",
                    modifier = Modifier
                        .background(Color.Yellow)
                        .wrapContentSize()
                )
            } else {
                Icon(imageVector = Icons.Filled.Face, contentDescription = "")
            }
        }
    }
}
```

### 5.4 修饰符 animateContentSize
使用 animateContentSize 为可组合项的大小变化添加动画效果。
**注意：animateContentSize 在修饰符链中的位置顺序很重要。为使动画流畅，请务必将其放在任何大小修饰符（如 size 或 defaultMinSize）之前，以确保 animateContentSize 向布局报告添加动画效果之后的值更改。**

示例：

```
@Composable
fun Demo() {
    var bigBox by remember {
        mutableStateOf(false)
    }
    Column {
        Button(onClick = {
            bigBox = !bigBox
        }) {
            Text(text = "Change Size")
        }
        Box(
            modifier = Modifier
                .background(Color.DarkGray)
                .animateContentSize()
                .size(if (bigBox) 200.dp else 50.dp)
        )
    }
}
```

效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic22.gif)

animateContentSize 定义如下：

```
fun Modifier.animateContentSize(
    animationSpec: FiniteAnimationSpec<IntSize> = spring(
        stiffness = Spring.StiffnessMediumLow
    ),
    finishedListener: ((initialValue: IntSize, targetValue: IntSize) -> Unit)? = null
): Modifier =
    this.clipToBounds() then SizeAnimationModifierElement(animationSpec, finishedListener)
```

两个参数，animationSpec 用来指定动画规格，finishedListener 用来监听动画结束。

## 六、特定场景使用动画
### 6.1 列表项的动画
传统视图中，使用 RecyclerView 可以为每一个列表项的更改添加动画效果，在 Compose 中，延迟布局 LazyXXX 提供了相同的功能。使用 `animateItemPlacement` 修饰符即可。

```
LazyColumn {
    items(books, key = { it.id }) {
        Row(Modifier.animateItemPlacement()) {
            // ...
        }
    }
}
```

也可以提供自定义的动画规格：

```
LazyColumn {
    items(books, key = { it.id }) {
        Row(
            Modifier.animateItemPlacement(
                tween(durationMillis = 250)
            )
        ) {
            // ...
        }
    }
}
```

使用时需要确保为子可组合项提供 key ，以便找到被移动的元素的新位置。
animateItemPlacement 提供了子元素重新排序动画，看起来好像很鸡肋。其实实际开发中更需要的增加或者移除某个子元素时的动画，很遗憾，目前还没有这样的 API, 目前正在开发用于添加或移除操作的可组合动画。

### 6.2 在 Navigation-Compose 导航中使用动画
当使用 Navigation-Compose 在不同的 Composable 之间导航时，可以在可组合项上指定 `enterTransition` 和 `exitTransition` 来实现动画效果，也可以在 `NavHost` 中设置用于所有目的地的默认动画。

```
val navController = rememberNavController()
NavHost(
    navController = navController, startDestination = "landing",
    enterTransition = { EnterTransition.None },
    exitTransition = { ExitTransition.None }
) {
    composable("landing") {
        ScreenLanding(
            // ...
        )
    }
    composable(
        "detail/{photoUrl}",
        arguments = listOf(navArgument("photoUrl") { type = NavType.StringType }),
        enterTransition = {
            fadeIn(
                animationSpec = tween(
                    300, easing = LinearEasing
                )
            ) + slideIntoContainer(
                animationSpec = tween(300, easing = EaseIn),
                towards = AnimatedContentTransitionScope.SlideDirection.Start
            )
        },
        exitTransition = {
            fadeOut(
                animationSpec = tween(
                    300, easing = LinearEasing
                )
            ) + slideOutOfContainer(
                animationSpec = tween(300, easing = EaseOut),
                towards = AnimatedContentTransitionScope.SlideDirection.End
            )
        }
    ) { backStackEntry ->
        ScreenDetails(
            // ...
        )
    }
}
```

目前 navigation-compose 还在不断迭代过程中，有些版本支持上述 enterTransition 和 exitTransition 设置动画，有些版本则没有相关 API。如果你当前使用的版本不支持，可以使用 Accompanist Navigation Animation 库。它提供了一套 API，帮助在 navigation-compose 中使用导航动画。
具体使用步骤：

```
implementation "com.google.accompanist:accompanist-navigation-animation:${version}"
```

- 使用 rememberAnimatedNavController() 替换rememberNavController()
- 使用 AnimatedNavHost 替换 NavHost
- 使用 import com.google.accompanist.navigation.animation.navigation 替换 import androidx.navigation.compose.navigation
- 使用 import com.google.accompanist.navigation.animation.composable 替换 import androidx.navigation.compose.composable

每个 composable 目的地都有四个新参数可以设置:

- enterTransition: 指定当您使用 navigate() 导航至该目的地时执行的动画。
- exitTransition: 指定当您通过导航至另一个目的地的方式离开该目的地时执行的动画。
- popEnterTransition: 指定当该目的地在经过调用 popBackStack() 后重新入场时执行的动画。默认为 enterTransition。
- popExitTransition: 指定当该目的地在以弹出返回栈的方式离开屏幕时执行的动画。默认为 exitTransition。

### 6.3 结合手势使用动画
除了单独处理 Composable 的动画外，还有一种常见的场景是动画和触摸事件需要同时处理。
可能需要考虑一下几点：

当触摸时间开始时，我们可能需要中断正在播放的动画，因为响应用户触摸应当具有最高优先级。
动画的值与触摸事件的值同步。
这里属于相对复杂的动画场景了。鉴于还没有讲解 Compose 手势相关的知识（后面计划再写一篇文章介绍 Compose 的手势处理），本文就不展开讲了，以后写自定义可组合函数的时候可能会有涉及。如想学习可参考官网讲解：
[高级动画实例：手势 https://developer.android.com/develop/ui/compose/animation/advanced?hl=zh-cn](https://developer.android.com/develop/ui/compose/animation/advanced?hl=zh-cn)

### 6.4 Compose 中的矢量图动画

#### 6.4.1 AnimatedVectorDrawable 文件格式
如需使用 AnimatedVectorDrawable 资源，请使用 animatedVectorResource 加载可绘制对象文件，并传入 boolean 以在可绘制对象的开始和结束状态之间切换，从而执行动画。

```
@Composable
fun AnimatedVectorDrawable() {
    val image = AnimatedImageVector.animatedVectorResource(R.drawable.ic_hourglass_animated)
    var atEnd by remember { mutableStateOf(false) }
    Image(
        painter = rememberAnimatedVectorPainter(image, atEnd),
        contentDescription = "Timer",
        modifier = Modifier.clickable {
            atEnd = !atEnd
        },
        contentScale = ContentScale.Crop
    )
}
```

#### 6.4.2 将 ImageVector 与 Compose 动画 API 结合使用
该场景不常用，本人还没仔细研究过。可参考一下文章，看起来效果不错。
[参考文章： Making Jellyfish move in Compose: Animating ImageVectors and applying AGSL RenderEffects](https://medium.com/androiddevelopers/making-jellyfish-move-in-compose-animating-imagevectors-and-applying-agsl-rendereffects-3666596a8888)

#### 6.4.3 使用 Lottie 动画
首先添加依赖：

```
dependencies {
    ...
    implementation "com.airbnb.android:lottie-compose:$lottieVersion"
    ...
}
```

基本使用方式：

```
@Composable
fun Loader() {
    val composition by rememberLottieComposition(LottieCompositionSpec.RawRes(R.raw.loading))
    val progress by animateLottieCompositionAsState(composition)
    LottieAnimation(
        composition = composition,
        progress = { progress },
    )
}
```

或者使用合并了 `LottieAnimation` 和 `animateLottieCompositionsState()` 的重载方法：

```
@Composable
fun Loader() {
    val composition by rememberLottieComposition(LottieCompositionSpec.RawRes(R.raw.loading))
    LottieAnimation(composition)
}
```

## 七、总结
经过前面的讲解，动画的知识点基本就都讲完了。内容还是有点多的，本节对 Compose 的动画做一个小结。常用 API 如何理解记忆，可见下图。

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic23.png)

高级别动画 API 底层实现都是基于低级别动画 API，提供了一个开箱即用的 `Composable` 或 `Modifier`。
如果需要更全面的 API，官方给了一个决策树图提供指导。

![](/assets/images/技术/编程/Jetpack%20Compose/compose6/pic24.jpg)