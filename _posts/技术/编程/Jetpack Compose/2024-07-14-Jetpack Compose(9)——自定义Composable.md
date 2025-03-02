---
layout: post
title: "Jetpack Compose(9)——自定义Composable"
date:  2024-07-14 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

## 一、Composable 组件渲染流程
传统 View 体系中，视图的渲染流程可以分为三个步骤： 测量、布局、绘制。在 Compose 中，组件的渲染流程也可划分为三步：组合、布局、绘制。

- 组合：执行Composable函数体，生成并维护LayoutNode视图树。
- 布局：对于视图树中的每个LayoutNode进行宽高尺寸测量并完成位置摆放。
- 绘制：将所有LayoutNode实际绘制到屏幕之上。

对于一般的组件都是正常经历 组合->布局->绘制 三个阶段来生成帧画面的，当然也存在特例，LazyColumn、LazyRow、等组件的子项合成可以延迟到这类组件的布局阶段进行，这是由于这类组件的子项组合需要依赖这类组件在布局阶段所能提供的一些信息。

### 1.1 组合
组合阶段的主要目标是生成并维护LayoutNode视图树，当我们在Activity中使用setContent时，会开始首次组合，此时会执行代码块中涉及的所有Composable函数体，生成与之对应的layoutNode视图树。与之相对应的是，在传统View系统中也是在setContentView中首次构建View视图树的。在Compose中如果Composable依赖了某个可变状态，当该状态发生更新时，会触发当前Composable重新进行组合阶段，故也被称为重组。在当前组件发生重组时，子Composable被依次重新调用：

- 被调用的子Composable将当前传入的参数与前次重组中的参数做比较，若参数变化，则Composable函数发生重组，更新LayoutNode视图树上对应节点，UI发生更新。
- 被调用的子Composable，对参数比较后，如果无任何变化，则跳过本次执行，即所谓的智能重组。LayoutNode视图树对应的节点保持不变，UI无变化。
- 如果子Composable在重组中没有再被调用到，其对应的节点及其子节点会从LayoutNode视图树中被删除，UI从屏幕移除。反之新增也是同理。

综上所述，重组过程可以自动维护LayoutNode视图树，使其永远保持在最新的视图状态。而在传统View系统中，只能手动对ViewGroup进行add/remove等操作来维护View视图树，这是两种视图体系本质的区别。
**注意：子Composable是否跳过重组除了取决于参数是否变化，也取决于参数类型是否是Stable。**

### 1.2 布局
布局阶段用来对视图树中每个LayoutNode进行宽高尺寸测量并完成位置摆放。当Compose的内置组件无法满足我们的需求时，可以在定制组件的布局阶段实现满足自己需求的组件。
在compose中，每个LayoutNode都会根据来自父LayoutNode的布局约束进行自我测量(类似传统View中的MeasureSpec)。布局约束中包含了父LayoutNode允许子LayoutNode的最大宽高和最小宽高，当父LayoutNode希望子LayoutNode测量的宽高为某个具体值时，约束中的最大宽高与最小宽高就是相同的。LayoutNode不允许被多次测量，在Compose中多次测量会抛异常。需要注意的是，有些需求场景需要多次测量LayoutNode，Compose为我们提供了固有特性测量与SubcomposeLayout作为解决方案。

### 1.3 绘制
绘制阶段主要是将所有LayoutNode实际绘制到屏幕之上，也可以对绘制阶段进行定制。

>理解了上面的流程，实际就可以对上面这三个渲染流程着手，进行自定义 Composable 组件。

## 二、自定义组合
在组合阶段自定义，理解起来最简单，实际上就是对 Composable 的封装。
我们知道 Composable 组件实际上是一个方法。我们根据自己需求，将不同的方法封装到一起，方便使用，就是这么个过程。

几个例子：想实现如下一个效果：
应用图标，有未读消息的时候，右上角显示一个小红点，并显示未读消息的数量，没有未读消息时，则不显示小红点。下面来实现这个需求：

```
@Composable
fun IconWithMsg(resId: Int, msgCount: Int) {
    Box(modifier = Modifier.size(60.dp)) {
        Image(
            painter = painterResource(id = resId),
            contentScale = ContentScale.FillBounds,
            contentDescription = null,
            modifier = Modifier
                .size(50.dp)
                .align(Alignment.Center)
        )
        // 根据状态不同，来组合不同的组件
        if (msgCount > 0) {
            Box(
                modifier = Modifier
                    .clip(CircleShape)
                    .background(Color.Red)
                    .size(25.dp)
                    .align(Alignment.TopEnd)
            ) {
                Text(
                    text = "$msgCount",
                    color = Color.White,
                    modifier = Modifier
                        .wrapContentSize()
                        .align(Alignment.Center)
                )
            }
        }
    }
}

@Composable
fun CustomTest() {
    Box(contentAlignment = Alignment.Center) {
        var msgCount by remember {
            mutableIntStateOf(0)
        }
        Column {
            IconWithMsg(R.mipmap.rc_3, msgCount)
            Spacer(modifier = Modifier.height(20.dp))
            Row {
                Button(onClick = { msgCount = 0 }) {
                    Text(text = "all read")
                }
                Spacer(modifier = Modifier.width(10.dp))
                Button(onClick = { msgCount++ }) {
                    Text(text = "add message")
                }
            }
        }
    }
}
```

效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose9/pic1.gif)

组件依赖于 msgCount 这个状态, msgCount 不为 0 时，则将新的 Composable 组合到一起。这也正是 Jetpack Compose 的重要思想。

从广义上讲，一切你封装的可复用的 Composable 组件，都可以认为是自定义的 Composable。并且从调用它的时机来看，就是在组合阶段。

## 三、自定义布局
### 3.1 LayoutModifier （自定义 View）

```
fun Modifier.customLayoutModifier(...) = Modifier.layout { measurable, constraints ->
    // measurable: child to be measured and placed
    // constraints: minimum and maximum for the width and height of the child
    ...
    // measure child
    val placeable = measurable.measure(constraints)
    // 设置view的width和height
    layout(width, height) {
        ...
        // 设置child显示的位置
        placeable.placeRelative(0, 0)
    }
}
```

例：实现设置Firstbaseline的padding功能:

```
@Composable
fun Modifier.firstBaselineTop(firstBaselineToTop: Dp) = this.then(
    layout { measurable, constraints ->
        val placeable = measurable.measure(constraints)
        check(placeable[FirstBaseline] != AlignmentLine.Unspecified)
        val firstBaseline = placeable[FirstBaseline]
        // 计算需要设置的padding值和原来的firstBaseline的差值
        val placeableY = firstBaselineToTop.roundToPx() - firstBaseline
        // View的高度（placeable.height为原来view的高度，包括padding）
        val height = placeable.height + placeableY

        layout(placeable.width, height) {
            placeable.placeRelative(
                0, placeableY
            )
        }
    }
)
```

继承自 LayoutModifier, 比如 Modifier.padding。

```
private class PaddingModifier(
    val start: Dp = 0.dp,
    val top: Dp = 0.dp,
    val end: Dp = 0.dp,
    val bottom: Dp = 0.dp,
    val rtlAware: Boolean,
) : LayoutModifier {

    override fun MeasureScope.measure(
        measurable: Measurable,
        constraints: Constraints
    ): MeasureResult {

        val horizontal = start.roundToPx() + end.roundToPx()
        val vertical = top.roundToPx() + bottom.roundToPx()

        val placeable = measurable.measure(constraints.offset(-horizontal, -vertical))

        val width = constraints.constrainWidth(placeable.width + horizontal)
        val height = constraints.constrainHeight(placeable.height + vertical)
        return layout(width, height) {
            if (rtlAware) {
                placeable.placeRelative(start.roundToPx(), top.roundToPx())
            } else {
                placeable.place(start.roundToPx(), top.roundToPx())
            }
        }
    }
}
```

### 3.2 Layout （自定义 ViewGroup）

```
@Composable
fun MyOwnColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // Don't constrain child views further, measure them with given constraints
        // List of measured children
        val placeables = measurables.map { measurable ->
            // Measure each child
            measurable.measure(constraints)
        }

        // child y方向的位置
        var yPosition = 0

        // Set the size of the layout as big as it can
        layout(constraints.maxWidth, constraints.maxHeight) {
            // Place children in the parent layout
            placeables.forEach { placeable ->
                // Position item on the screen
                placeable.placeRelative(x = 0, y = yPosition)

                // 累加当前view的高度
                yPosition += placeable.height
            }
        }
    }
}
```

### 3.3 固有特性测量Intrinsic
前面我们提到了Compose布局原理，在Compose中的每个LayoutNode是不允许被多次进行测量的，多次测量在运行时会抛异常，但在很多场景中多次测量子UI组件是有意义的。假设有这样的需求场景，希望中间分割线与两边文案的一侧等高。

为实现这个需求，假设可以预先测量得到两边文案组件的高度信息，取其中的最大值作为当前组件的高度，此时仅需将分割线高度值铺满整个父组件即可。

固有特性测量为我们提供了预先测量所有子组件来确定自身constraints（布局约束）的能力，并在正式测量阶段对子组件的测量产生影响。

#### 3.3.1 使用内置组件的固有特性测量
使用固有特性测量的前提是组件需要适配固有特性测量，目前许多内置组件已经实现了固有特性测量，可以直接使用。绝大多数内置组件都是用LayoutComposable实现的，LayoutComposable中需要传入一个measurePolicy，默认只需实现measure，但如果要实现固有特性测量，就需要额外重写Intrinsic系列方法。
父组件所提供的能力使用基础组件中的Row组件即可承担，仅需为Row组件高度设置固有特性测量即可。使用Modifier.height(IntrinsicSize. Min)即可为高度设置固有特性测量。

```
@Composable
fun TwoTexts(modifier: Modifier = Modifier, text1: String, text2: String) {
    Row(modifier = modifier.height(IntrinsicSize.Min)) {   //这里使用了固有特性测量
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(start = 4.dp)
                .wrapContentWidth(Alignment.Start),
            text = text1,
            fontSize = 16.sp    //字体大小不一样,高度不一样
        )

        Divider(color = Color.Black, modifier = Modifier.fillMaxHeight().width(1.dp))  //高度为父布局最大高度
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(end = 4.dp)
                .wrapContentWidth(Alignment.End),
            text = text2,
            fontSize = 30.sp    //字体大小不一样,高度不一样
        )
    }
}

//使用
Surface(border = BorderStroke(1.dp, Color.Blue) ) {
    TwoTexts(text1 = "Hi,kotlin", text2 = "Hello,World")
}
```

值得注意的是，此时仅使用Modifier.height(IntrinsicSize. Min)为高度设置了固有特性测量，宽度并没有进行设置，此时表示当宽度不限定时，根据子组件预先测量的宽高信息来计算父组件高度所允许的最小值。当然也可以设置宽度，也就表示当宽度受到限制时，根据子组件测量的宽高信息来计算父组件的宽度所允许的最小值。
我们只能对已经适配固有特性测量的内置组件使用IntrinsicSize. Min或IntrinsicSize. Max，否则程序运行时会crash。

#### 3.3.2 自定义固有特性测量
在上面的例子中，使用Row组件的固有特性测量，预先测量子组件，并根据子组件的高度来确定Row组件的高度。然而其中具体是如何操作的，答案都藏在Row组件源码中。前面也提到如果想适配固有特性测量，需要额外重写measurePolicy中的固有特性测量Intrinsic系列方法。
打开MeasurePolicy的接口声明，我们看到Intrinsic系列方法共有四个，如下所示：

```
@Stable
@JvmDefaultWithCompatibility
fun interface MeasurePolicy {

    fun MeasureScope.measure(measurables: List<Measurable>,constraints: Constraints): MeasureResult

    fun IntrinsicMeasureScope.minIntrinsicWidth(measurables: List<IntrinsicMeasurable>,height: Int): Int {
        ...
    }

    fun IntrinsicMeasureScope.minIntrinsicHeight(measurables: List<IntrinsicMeasurable>,width: Int): Int {
        ...
    }

    fun IntrinsicMeasureScope.maxIntrinsicWidth(measurables: List<IntrinsicMeasurable>,height: Int): Int {
        ...
    }

    fun IntrinsicMeasureScope.maxIntrinsicHeight(measurables: List<IntrinsicMeasurable>,width: Int): Int {
        ...
    }
}
```

当使用Modifier.width(IntrinsicSize. Max)时，在测量阶段便会调用maxIntrinsicWidth方法，以此类推。
在使用固有特性测量前，需要确定对应Intrinsic方法是否重写，如果没有重写，则会crash。既然要实现Intrinsic方法，在Layout声明时就不能简单使用SAM转换了，需要规规矩矩实现MeasurePolicy接口，如下：

```
@Composable
fun MyOwnColumn(modifier: Modifier = Modifier, content: @Composable () -> Unit) {
    Layout(modifier = modifier, content = content, measurePolicy = object :MeasurePolicy{
        override fun MeasureScope.measure(
            measurables: List<Measurable>,
            constraints: Constraints
        ): MeasureResult {
            TODO("Not yet implemented")
        }

        override fun IntrinsicMeasureScope.maxIntrinsicHeight(
            measurables: List<IntrinsicMeasurable>,
            width: Int
        ): Int {
            TODO("Not yet implemented")
        }

        override fun IntrinsicMeasureScope.maxIntrinsicWidth(
            measurables: List<IntrinsicMeasurable>,
            height: Int
        ): Int {
            TODO("Not yet implemented")
        }

        override fun IntrinsicMeasureScope.minIntrinsicHeight(
            measurables: List<IntrinsicMeasurable>,
            width: Int
        ): Int {
            TODO("Not yet implemented")
        }

        override fun IntrinsicMeasureScope.minIntrinsicWidth(
            measurables: List<IntrinsicMeasurable>,
            height: Int
        ): Int {
            TODO("Not yet implemented")
        }
    })
}
```

下面我们自定义Row组件实现与官方Row组件同样的固有特性效果。因为我们的需求场景只使用了Modifier.height(IntrinsicSize. Min)，所以仅重写minIntrinsicHeight方法就可以了。

在重写的minIntrinsicHeight方法中，可以拿到子组件预先测量句柄intrinsicMeasurables。这个与前面提到的measurables用法完全相同。在预先测量所有子组件后，就可以根据子组件的高度计算其中的高度最大值，此值将会影响到正式测量时父组件获取到的constraints的高度信息。此时constraints中的maxHeight与minHeight都将被设置为返回的高度值，constraints中的高度为一个确定值：

```
override fun IntrinsicMeasureScope.minIntrinsicHeight(
    measurables: List<IntrinsicMeasurable>,
    width: Int
): Int {
    var maxHeight = 0
    measurables.forEach {
        //找出最大的高度并赋值给maxHeight
        maxHeight = it.minIntrinsicHeight(width).coerceAtLeast(maxHeight)
    }
    return maxHeight
}
```

接着我们在测量中直接左右摆放，代码如下：

```
override fun MeasureScope.measure(
    measurables: List<Measurable>,
    constraints: Constraints
): MeasureResult {
    val placeables = measurables.map { measurable ->
        //给控件设置layoutId可以直接找到控件,自定义测量规则
        //measurable.layoutId=="Divider"
        measurable.measure(constraints)
    }
    var positionX = 0
    return layout(constraints.maxWidth, constraints.maxHeight) {
        placeables.forEach { placeable ->
            placeable.placeRelative(positionX, 0)
            positionX += placeable.width
        }
    }
}
```

接着使用我们的固有测量属性，完整代码如下：

```
@Composable
fun Greeting() {
    MyOwnRow(modifier = Modifier.height(IntrinsicSize.Min)) {   //使用了自定义的固有测量属性
        Text(
            text = "Hello Kotlin",
            fontSize = 10.sp    //字体大小不一样,高度不一样
        )

        Divider(
            color = Color.Black, modifier = Modifier
                .fillMaxHeight()
                .width(1.dp)
            //.layoutId("Divider")
        )

        Text(
            text = "Hello Android",
            fontSize = 18.sp    //字体大小不一样,高度不一样
        )
    }

}

@Composable
fun MyOwnRow(modifier: Modifier = Modifier, content: @Composable () -> Unit) {
    Layout(modifier = modifier, content = content, measurePolicy = object : MeasurePolicy {
        override fun MeasureScope.measure(
            measurables: List<Measurable>,
            constraints: Constraints
        ): MeasureResult {
            val placeables = measurables.map { measurable ->
                //给控件设置layoutId可以直接找到控件,自定义测量规则
                //measurable.layoutId=="Divider"
                measurable.measure(constraints)
            }
            var positionX = 0
            return layout(constraints.maxWidth, constraints.maxHeight) {
                placeables.forEach { placeable ->
                    placeable.placeRelative(positionX, 0)
                    positionX += placeable.width
                }
            }
        }

        override fun IntrinsicMeasureScope.minIntrinsicHeight(
            measurables: List<IntrinsicMeasurable>,
            width: Int
        ): Int {
            var maxHeight = 0
            measurables.forEach {
                //找出最大的高度并赋值给maxHeight
                maxHeight = it.minIntrinsicHeight(width).coerceAtLeast(maxHeight)
            }
            return maxHeight
        }
    })
}
```

固有特性测量的本质就是允许父组件预先获取到每个子组件宽高信息后，影响自身在测量阶段获取到的constraints宽高信息，从而间接影响子组件的测量过程。在上面的例子中我们通过预先测量文案子组件的高度，从而确定了父组件在测量时获取到的constraints高度信息，并根据这个高度指定了分割线高度。

### 3.4 SubcomposeLayout
SubcomposeLayout允许子组件的组合阶段延迟到父组件的布局阶段进行，为我们提供了更强的测量定制能力。前面曾提到，固有特性测量的本质就是允许父组件预先获取到每个子组件宽高信息后，影响自身在测量阶段获取到的constraints宽高信息，从而间接影响子组件的测量过程。而利用SubcomposeLayout，可以做到将某个子组件的组合阶段延迟至其所依赖的同级子组件测量结束后进行，从而可以定制子组件间的组合、布局阶段顺序，以取代固有特性测量。

我们使用SubcomposeLayout实现上面固有特性中的例子。
在上面固有特性测量的例子中，可以先测量两侧文本的高度，而后为Divider指定高度，再进行测量。与固有特性测量不同的是，在整个过程中父组件是没有参与的。接下来看看SubcomposeLayout组件是如何使用的。

```
@Composable
@UiComposable
fun SubcomposeLayout(
    state: SubcomposeLayoutState,
    modifier: Modifier = Modifier,
    measurePolicy: SubcomposeMeasureScope.(Constraints) -> MeasureResult
) {...}
```

- `state` 用于跟踪子组合的状态的对象，包括已分配的标签和测量信息。通过此参数，可以访问或修改子组合的状态。
- `measurePolicy` 一个 lambda 函数，定义了子组合的测量策略。此函数接收 SubcomposeMeasureScope 和 Constraints 作为参数，返回 MeasureResult 对象，表示子组合的测量结果。SubcomposeMeasureScope 提供一些用于测量的便捷函数和属性，而 Constraints 描述了父布局对子组合的测量要求。

其实SubcomposeLayout和Layout组件是差不多的。不同的是，此时需要传入一个SubcomposeMeasureScope类型Lambda，打开接口声明可以看到其中仅有一个（名为subcompose）。

```
interface SubcomposeMeasureScope : MeasureScope {
    fun subcompose(slotId: Any?, content: @Composable () -> Unit): List<Measurable>
}
```

subcompose会根据传入的slotId和Composable生成一个LayoutNode用于构建子Composition，最终会返回所有子LayoutNode的Measurable测量句柄。其中Composable是我们声明的子组件信息。slotId是用来让SubcomposeLayout追踪管理我们所创建的子Composition的，作为唯一索引每个Composition都需要具有唯一的slotId，接下来看看如何在前面的示例场景中使用SubcomposeLayout。
实际上可以把所有待测量的组件分为文字组件和分隔符组件两部分。由于分隔符组件的高度是依赖文字组件的，所以声明分隔符组件时传入一个Int值作为测量高度。首先定义一个Composable。

```
@Composable
fun SubcomposeRow(
    modifier: Modifier,
    text: @Composable () -> Unit,
    divider: @Composable (Int) -> Unit//传入高度
) {
    SubcomposeLayout(modifier = modifier) { constraints ->
        ...
    }
}
```

首先可以使用subcompose来测量text中的所有LayoutNode，并根据测量结果计算出最大高度。

```
@Composable
fun SubcomposeRow(
    modifier: Modifier,
    text: @Composable () -> Unit,
    divider: @Composable (Int) -> Unit//传入高度
) {
    SubcomposeLayout(modifier = modifier) { constraints ->
       var maxHeight=0
        var placeables = subcompose(slotId = "text",text).map {
            val placeable = it.measure(constraints)
            maxHeight = placeable.height.coerceAtLeast(maxHeight)
            placeable
        }
            ...
    }
}
```

既然计算得到了文本的最大高度，接下来就可以将高度只传入分隔符组件中，完成组合阶段并进行测量。

```
@Composable
fun SubcomposeRow(
    modifier: Modifier,
    text: @Composable () -> Unit,
    divider: @Composable (Int) -> Unit//传入高度
) {
    SubcomposeLayout(modifier = modifier) { constraints ->
       var maxHeight=0
        var placeables = subcompose(slotId = "text",text).map {
            val placeable = it.measure(constraints)
            maxHeight = placeable.height.coerceAtLeast(maxHeight)
            placeable
        }
          val dividerPlaceable = subcompose(slotId ="divider"){
              divider(maxHeight)
          }.map {
              it.measure(constraints.copy(minWidth = 0))
          }
              ...
    }
}
```

与前面固有特性测量中的一样，在测量Divider组件时，仍需重新复制一份constraints并将其minWidth设置为0，如果不修改，Divider组件宽度默认会与整个组件宽度相同。接下来分别对文字组件和分隔符组件进行布局。

```
@Composable
fun SubcomposeRow(
    modifier: Modifier,
    text: @Composable () -> Unit,
    divider: @Composable (Int) -> Unit//传入高度
) {
    SubcomposeLayout(modifier = modifier) { constraints ->
        var maxHeight = 0
        var placeables = subcompose(slotId = "text", text).map {
            val placeable = it.measure(constraints)
            maxHeight = placeable.height.coerceAtLeast(maxHeight)
            placeable
        }
        // divider设置文本最大的高度
        val dividerPlaceable = subcompose(slotId = "divider") {
            divider(maxHeight)
        }.map {
            it.measure(constraints.copy(minWidth = 0))
        }
        //布局
        layout(constraints.maxWidth, constraints.maxHeight) {
            placeables.forEach {
                it.placeRelative(0, 0)
            }
            val midPos = constraints.maxWidth / 2
            dividerPlaceable.forEach {
                it.placeRelative(midPos, 0)
            }
        }
    }
}
```

SubcomposeRow控件的使用

```
@Composable
fun Greeting() {
    SubcomposeRow(modifier = Modifier.fillMaxWidth(), text = {
        Text(
            text = "Hello Kotlin",
            fontSize = 14.sp,    //字体大小不一样,高度不一样
            modifier = Modifier.wrapContentWidth(Alignment.Start)
        )
        Text(
            text = "Hello Android",
            fontSize = 18.sp,    //字体大小不一样,高度不一样
            modifier = Modifier.wrapContentWidth(Alignment.End)
        )

    }) { heightPx ->   //使用高度
        //px转dp
        val heightDp = with(LocalDensity.current) {
            heightPx.toDp()
        }
        Divider(
            color = Color.Black, modifier = Modifier
                .height(heightDp)   //使用高度
                .width(1.dp)
        )
    }
}
```

SubcomposeLayout具有更强的灵活性，然而性能上不如常规Layout，因为子组件的组合阶段需要延迟到父组件布局阶段才能进行，因此还需要额外创建一个子Composition，因此SubcomposeLayout可能并不适用在一些对性能要求比较高的UI部分。

## 四、自定义绘制
绘制阶段主要是将所有LayoutNode实际绘制到屏幕之上，也可以对绘制阶段进行定制。如果我们对Android原生Canvas已经非常熟悉，迁移到Compose是没有任何学习成本的。

### 4.1 Canvas Composable
Canvas Composable是官方提供的一个专门用来自定义绘制的单元组件。CanvasComposable包含两个参数，一个是Modifier，另一个是DrawScope作用域代码块。

```
@Composable
fun Canvas(modifier: Modifier, onDraw: DrawScope.() -> Unit) =
    Spacer(modifier.drawBehind(onDraw))   //所有的绘制逻辑最终都传入drawBehind()修饰符方法里
```

在DrawScope作用域中，Compose提供了基础绘制API，如表所示:

|API|描述|
|---|---|
|drawLine|绘制线|
|drawRect|绘制矩形|
|drawlmage|绘制图片|
|drawRoundRect|绘制圆角矩形|
|drawCircle|绘制圆|
|drawOval|绘制椭圆|
|drawArc|绘制弧线|
|drawPath|绘制路径|
|drawPoints|绘制点|

示例：

```
@Preview
@Composable
fun DrawColorRing() {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        var radius = 300.dp
        var ringWidth = 30.dp
        Canvas(modifier = Modifier.size(radius)) {
            this.drawCircle( // 画圆
                brush = Brush.sweepGradient(listOf(Color.Red, Color.Green, Color.Red), Offset(radius.toPx() / 2f, radius.toPx() / 2f)),
                radius = radius.toPx() / 2f,
                style = Stroke(
                    width = ringWidth.toPx()
                )
            )
        }
    }
}
```

![](/assets/images/技术/编程/Jetpack%20Compose/compose9/pic2.png)

### 4.2 DrawModifier
DrawModifier类修饰符方法共有三个，每个都有其各自的用途。drawWithContent允许开发者可以在绘制时自定义绘制层级，Canvas Composable中使用的drawBehind是用来定制绘制组件背景的，而drawWithCache则允许开发者在绘制时可以携带缓存。

#### 4.2.1 drawWithContent
在drawBehind的实现源码中，定制的绘制逻辑onDraw会被传入DrawBackgroundModifier的主构造器中。在重写的 draw 方法中，首先调用了我们传入的定制绘制逻辑，之后调用drawContent来绘制组件内容本身。

```
fun Modifier.drawBehind(
    onDraw: DrawScope.() -> Unit
) = this then DrawBehindElement(onDraw)    //DrawBehindElement

@OptIn(ExperimentalComposeUiApi::class)
private data class DrawBehindElement(   //DrawBehindElement
    val onDraw: DrawScope.() -> Unit
) : ModifierNodeElement<DrawBackgroundModifier>() {
    override fun create() = DrawBackgroundModifier(onDraw)  //DrawBackgroundModifier

   ...
}

@OptIn(ExperimentalComposeUiApi::class)
private class DrawBackgroundModifier(    //DrawBackgroundModifier
    var onDraw: DrawScope.() -> Unit
) : Modifier.Node(), DrawModifierNode {

    override fun ContentDrawScope.draw() {
        onDraw()    //先绘制背景
        drawContent()   //再绘制内容
    }
}
```

#### 4.2.2 drawBehind
drawBehind，画在后面。具体画在谁后面呢，具体画在他所修饰的UI组件后面。根据前面的介绍，我们就可以猜到，其实不就是先画我们自己定制的绘制控制逻辑后，再画UI组件本身嘛？我们翻阅源码可以看到。

```
fun Modifier.drawBehind(
    onDraw: DrawScope.() -> Unit
) = this.then(
    DrawBackgroundModifier(
        onDraw = onDraw, // onDraw 为我们定制的绘制控制逻辑
        ...
    )
)

private class DrawBackgroundModifier(
    val onDraw: DrawScope.() -> Unit,
    ...
) : DrawModifier, InspectorValueInfo(inspectorInfo) {
    override fun ContentDrawScope.draw() {
        onDraw() // 先画我们定制的绘制控制逻辑
        drawContent() // 后画UI组件本身
    }
    ...
}
```

#### 4.2.3 drawWithCache
有些时候我们绘制一些比较复杂的UI效果时，不希望当 Recompose 发生时所有绘画所用的所有实例都重新构建一次（类似Path），这可能会产生内存抖动。在 Compose 中我们一般能够想到使用 remember 进行缓存，然而我们所绘制的作用域是 DrawScope 并不是 Composable，所以无法使用 remember，drawWithCache 提供了这个能力。

打开 drawWithCache 的声明可以看到，需要传入一个Reciever为 CacheDrawScope 类型的lambda，值得注意的是此时返回值必须是一个 DrawResult。接下来我们看看 CacheDrawScope 为我们限定了哪些 API。

哈哈可以看到，CacheDrawScope 中的 onDrawBehind、onDrawWithContent 提供了 DrawResult 类型返回值，这两个 API 完全等价于 drawBehind 与drawWithContent。怎么用就不必多说了。

```
@Preview
@Composable
fun DrawBorder() {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            var borderColor by mutableStateOf(Color.Red)
            Card(
                shape = RoundedCornerShape(0.dp)
                ,modifier = Modifier
                    .size(100.dp)
                    .drawWithCache {
                        Log.d("compose_study", "此处不会发生 Recompose")
                        var path = Path().apply {
                            moveTo(0f, 0f)
                            relativeLineTo(100.dp.toPx(), 0f)
                            relativeLineTo(0f, 100.dp.toPx())
                            relativeLineTo(-100.dp.toPx(), 0f)
                            relativeLineTo(0f, -100.dp.toPx())
                        }
                        onDrawWithContent {
                            Log.d("compose_study", "此处会发生 Recompose")
                            drawContent()
                            drawPath(
                                path = path,
                                color = borderColor,
                                style = Stroke(
                                    width = 10f,
                                )
                            )
                        }
                    }
            ) {
                Image(painter = painterResource(id = R.drawable.diana), contentDescription = "Diana")
            }
            Spacer(modifier = Modifier.height(20.dp))
            Button(onClick = {
                borderColor = Color.Yellow
            }) {
                Text("Change Color")
            }
        }
    }
}
```

### 4.3 与原生兼容
DrawScope 中所提供的 API 仅是一个高层次的封装，底层仍然是用的是原生平台的 Canvas 进行绘制。作为一个高层次封装，为了保证平台通用性，必然会导致具体平台 API 提供的一些 API 的丢失。例如，我们在 Android 原生 Canvas 可以绘制文字 drawText，但这在 DrawScope 是没有被提供的。
在 DrawScope 中，我们可以访问到 drawContext 成员，drawContext 存储了以下信息。

- size： 绘制尺寸
- canvas： Compose 封装的高层次 Canvas
- transform： transform控制器，用以旋转、缩放与移动

可以通过 canvas.nativeCanvas 获取具体平台 Canvas 实例，在 Android 平台就对应AndroidCanvas，通过这个 nativeCanvas 就可以调用到原生平台 Canvas 方法了。所以如果你不喜欢使用 DrawScope 提供的平台通用 API或是需求需要，可以直接使用原生平台 Canvas ，但这样做的代价就是会丢失平台通用性，对于不同平台需要给予不同的实现，不能作为一个通用模块进行提供，如果你只针对 Android 平台进行开发就不需要考虑这么多了。