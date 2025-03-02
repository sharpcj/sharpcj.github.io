---
layout: post
title: "Jetpack Compose(8)——嵌套滚动"
date:  2024-06-28 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

## 前言
所谓嵌套滚动，就是两个组件之间出现滚动事件冲突了，要给与特定的处理逻辑。在传统 View 系统中称之为滑动冲突，一般有两种解决方案，外部拦截法和内部拦截法。在 Jetpack Compose 中，提供了 Modifier.nestedScroll 修饰符用来处理嵌套滚动的场景。

## 一、Jetpack Compose 中处理嵌套滚动的思想
在介绍 `Modifier.nestedScroll` 之前，需要先了解 Compose 中嵌套滚动的处理思想。当组件获得滚动事件后，先交给它的父组件消费，父组件消费之后，将剩余可用的滚动事件在给到子组件，子组件再消费，子组件消费之后，再将剩余的滚动事件再给到父组件。

第一趟 ... -> 孙 ——> 子 ——> 父 -> ...
第二趟 ... <- 孙 <—— 子 <—— 父 <- ...
第三趟 ... -> 孙 ——> 子 ——> 父 -> ...

## 二、Modifier.nestedScroll
有了整体思路之后，再来看 Modifier.nestedScroll 这个修饰符。

```
fun Modifier.nestedScroll(
    connection: NestedScrollConnection,
    dispatcher: NestedScrollDispatcher? = null
): Modifier
```

使用 nestedScroll 参数列表中有一个必选参数 `connection` 和一个可选参数 `dispatcher`

- connection: 嵌套滑动手势处理的核心逻辑，内部回调可以在子布局获得滑动事件前预先消费掉部分或全部手势偏移量，也可以获取子布局消费后剩下的手势偏移量。
- dispatcher：调度器，内部包含用于父布局的 NestedScrollConnection , 可以调用 dispatch* 方法来通知父布局发生滑动

### 2.1 NestedScrollConnection
`NestedScrollConnection` 提供了四个回调方法。

```
interface NestedScrollConnection {
    /**
    * 预先劫持滑动事件，消费后再交由子布局。
    * available：当前可用的滑动事件偏移量
    * source：滑动事件的类型
    * 返回值：当前组件消费的滑动事件偏移量，如果不想消费可返回Offset.Zero
    */
    fun onPreScroll(available: Offset, source: NestedScrollSource): Offset = Offset.Zero

    /**
    * 获取子布局处理后的滑动事件
    * consumed：之前消费的所有滑动事件偏移量
    * available：当前剩下还可用的滑动事件偏移量
    * source：滑动事件的类型
    * 返回值：当前组件消费的滑动事件偏移量，如果不想消费可返回 Offset.Zero ，则剩下偏移量会继续交由当前布局的父布局进行处理
    */
    fun onPostScroll(
        consumed: Offset,
        available: Offset,
        source: NestedScrollSource
    ): Offset = Offset.Zero

    /**
     * 获取 Fling 开始时的速度
     * available：Fling 开始时的速度
     * 返回值：当前组件消费的速度，如果不想消费可返回 Velocity.Zero
     */
    suspend fun onPreFling(available: Velocity): Velocity = Velocity.Zero

    /**
     * 获取 Fling 结束时的速度信息
     * consumed：之前消费的所有速度
     * available：当前剩下还可用的速度
     * 返回值：当前组件消费的速度，如果不想消费可返回Velocity.Zero，剩下速度会继续交由当前布局的父布局进行处理
     */
    suspend fun onPostFling(consumed: Velocity, available: Velocity): Velocity {
        return Velocity.Zero
    }
}
```

关于各个方法的含义已经在方法注释中标注了。

**注意 Fling 的含义：** 当我们手指在滑动列表时，如果是快速滑动并抬起，则列表会根据惯性继续飘一段距离后停下，这个行为就是 Fling ，onPreFling 在你手指刚抬起时便会回调，而 onPostFling 会在飘一段距离停下后回调。

### 2.2 NestedScrollDispatcher
NestedScrollDispatcher 的主要方法：

```
fun dispatchPreScroll(available: Offset, source: NestedScrollSource): Offset {
    return parent?.onPreScroll(available, source) ?: Offset.Zero
}

fun dispatchPostScroll(
        consumed: Offset,
        available: Offset,
        source: NestedScrollSource
    ): Offset {
        return parent?.onPostScroll(consumed, available, source) ?: Offset.Zero
    }

suspend fun dispatchPreFling(available: Velocity): Velocity {
    return parent?.onPreFling(available) ?: Velocity.Zero
}

suspend fun dispatchPostFling(consumed: Velocity, available: Velocity): Velocity {
    return parent?.onPostFling(consumed, available) ?: Velocity.Zero
}
```

其实方法实现就能清楚，实际上让其父组件调用 `NestedScrollConnection` 中的预消费与后消费方法。

## 三、实操讲解
### 3.1 父组件消费子组件给过来的事件——NestedScrollConnection
先上效果图：

![](/assets/images/技术/编程/Jetpack%20Compose/compose8/pic1.gif)

简单分析一下效果：

1. 布局分为两部分，上面是一张图片，下面是一个滑动列表
2. 滑动过程中，上滑时，首先头部响应滑动，收缩到最小高度之后，列表再开始向上滑动。下滑时，也是头部先影响滑动，头部图片展开到最大高度之后，列表再开始向下滑动。即：不论上上滑还是下滑，都是头部图片先响应。
3. 我们希望是按住列表能滑动，按住头部图片是不能滑动的，也就是说头部图片不会检测滑动事件，只有下面列表会检测滑动事件。

下面开始编码：

1. 手写整体布局应该是 Column 实现，头部使用一个 Image ，下面使用 LazyColumn

```
@Composable
fun NestedScrollDemo() {
    Column(
        modifier = Modifier.fillMaxSize()) {
            Image(
                painter = painterResource(id = R.mipmap.rc_1),
                contentDescription = null,
                contentScale = ContentScale.FillBounds,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(200.dp)
            )

            LazyColumn {
                repeat(50) {
                    item {
                        Text(text = "item --> $it", modifier = Modifier.fillMaxWidth())
                    }
                }
            }
    }
}
```

2. 给 Column 组件使用 Modifier.nestedScroll。
这里简单做一些定义：头部图片最小高度为 80.dp， 最大高度为 200.dp。注意 dp 和 px 之间的转换。

```
@Composable
fun NestedScrollDemo() {
    val minHeight = 80.dp
    val maxHeight = 200.dp
    val density = LocalDensity.current

    val minHeightPx = with(density) {
        minHeight.toPx()
    }

    val maxHeightPx = with(density) {
        maxHeight.toPx()
    }

    var topHeightPx by remember {
        mutableStateOf(maxHeightPx)
    }

    val connection = remember {
        object : NestedScrollConnection {
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                return super.onPreScroll(available, source)
            }

            override fun onPostScroll(consumed: Offset, available: Offset, source: NestedScrollSource): Offset {
                return super.onPostScroll(consumed, available, source)
            }

            override suspend fun onPreFling(available: Velocity): Velocity {
                return super.onPreFling(available)
            }

            override suspend fun onPostFling(consumed: Velocity, available: Velocity): Velocity {
                return super.onPostFling(consumed, available)
            }
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .nestedScroll(connection = connection)
    ) {
        Image(
            painter = painterResource(id = R.mipmap.rc_1),
            contentDescription = null,
            contentScale = ContentScale.FillBounds,
            modifier = Modifier
                .fillMaxWidth()
                .height(with(density) {
                    topHeightPx.toDp()
                })
        )

        LazyColumn {
            repeat(50) {
                item {
                    Text(text = "item --> $it", modifier = Modifier.fillMaxWidth())
                }
            }
        }
    }
}
```

3. 最后就是编写滑动处理逻辑了。LazyColumn 列表检测到滑动事件，把这个滑动距离先给到父组件Column 消费，Column 消费之后，把剩余的再给到 LazyColumn 消费，LazyColumn 消费之后，还有剩余，再给回 Column 消费。其中 LazyColumn 消费事件，不用我们处理，我们的 Modifier.nestedScroll 作用在 Column 上，我们需要预先消费 LazyColumn 给过来的滑动距离——在 `onPreScroll` 中实现，然后把剩余的给到 LazyColumn，最后 LazyColumn 消费后还有剩余的滑动距离，Column 处理 —— 在 `onPostScroll` 中处理。

```
val connection = remember {
    object : NestedScrollConnection {
        /**
         * 预先劫持滑动事件，消费后再交由子布局。
         * available：当前可用的滑动事件偏移量
         * source：滑动事件的类型
         * 返回值：当前组件消费的滑动事件偏移量，如果不想消费可返回Offset.Zero
         */
        override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
            if (source == NestedScrollSource.Drag) {  // 判断是滑动事件
                if (available.y < 0) { // 向上滑动
                    val dH = minHeightPx - topHeightPx  // 向上滑动过程中，还差多少达到最小高度
                    if (available.y > dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最小高度，就将当前可用距离全部消费掉
                        topHeightPx += available.y
                        return Offset(x = 0f, y = available.y)
                    } else {  // 如果当前可用的滑动距离足够达到最小高度，就只消费掉需要的距离。剩余的给到子组件。
                        topHeightPx += dH
                        return Offset(x = 0f, y = dH)
                    }
                } else { // 下滑
                    val dH = maxHeightPx - topHeightPx  // 向下滑动过程中，还差多少达到最大高度
                    if (available.y < dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最大高度，就将当前可用距离全部消费掉
                        topHeightPx += available.y
                        return Offset(x = 0f, y = available.y)
                    } else {  // 如果当前可用的滑动距离足够达到最大高度，就只消费掉需要的距离。剩余的给到子组件。
                        topHeightPx += dH
                        return Offset(x = 0f, y = dH)
                    }
                }
            } else {  // 如果不是滑动事件，就不消费。
                return Offset.Zero
            }
        }

        /**
         * 获取子布局处理后的滑动事件
         * consumed：之前消费的所有滑动事件偏移量
         * available：当前剩下还可用的滑动事件偏移量
         * source：滑动事件的类型
         * 返回值：当前组件消费的滑动事件偏移量，如果不想消费可返回 Offset.Zero ，则剩下偏移量会继续交由当前布局的父布局进行处理
         */
        override fun onPostScroll(
            consumed: Offset, available: Offset, source: NestedScrollSource
        ): Offset {
            // 子组件处理后的剩余的滑动距离，此处不需要消费了，直接不消费。
            return Offset.Zero
        }

        override suspend fun onPreFling(available: Velocity): Velocity {
            return super.onPreFling(available)
        }

        override suspend fun onPostFling(consumed: Velocity, available: Velocity): Velocity {
            return super.onPostFling(consumed, available)
        }
    }
}
```

所有的代码通过注释已经写的很详细了。这样就实现了上图的效果。

### 3.2 子组件对事件进行分发——NestedScrollDispatcher
滑动距离消费不一定要体现在位置、大小之类的变化上。当使用 Modifier.nestedScroll 修饰符处理嵌套滚动时，绝大多数场景使用外部拦截法就能轻松实现，给父容器修饰，实现 NestedScollConnection 方法。

使用内部拦截法,一般用于父组件也可以消费事件，需要子容器使用 Modifier.nestedScroll ，并合理使用 NestedScrollDispatcher 的方法。

看下面这个示例

![](/assets/images/技术/编程/Jetpack%20Compose/compose8/pic2.gif)

同样简单分析一下效果：

1. 整个父组件是一个 LazyColumn， 自身可以滚动
2. LazyColumn 中的一个元素是一张图片，使用 Image 组件，当按住图片滚动时，优先处理图片的收缩与展开。
实现如下：

```
@Composable
fun NestedScrollDemo4() {
    val minHeight = 80.dp
    val maxHeight = 200.dp
    val density = LocalDensity.current

    val minHeightPx = with(density) {
        minHeight.toPx()
    }

    val maxHeightPx = with(density) {
        maxHeight.toPx()
    }

    var topHeightPx by remember {
        mutableStateOf(maxHeightPx)
    }

    val connection = remember {
        object : NestedScrollConnection {}
    }

    val dispatcher = remember { 
        NestedScrollDispatcher() 
    }

    LazyColumn(
        modifier = Modifier
            .background(Color.LightGray)
            .fillMaxSize()
    ) {
        for (i in 0..10) {
            item {
                Text(text = "item --> $i", modifier = Modifier.fillMaxWidth())
            }
        }
        item {
            Image(
                painter = painterResource(id = R.mipmap.rc_1),
                contentDescription = null,
                contentScale = ContentScale.FillBounds,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(with(density) {
                        topHeightPx.toDp()
                    })
                    .draggable(
                        state = rememberDraggableState { onDelta ->
                            // 1. 滑动距离，给到父组件先消费
                            // 调用父组件劫持滑动事件，让父组件先消费，返回值是父组件消费掉的滑动距离
                            // 这里并不想让父组件先消费，就给父组件传了 Offset.Zero。 返回值也就是 Offset.Zero。
                            val consumed = dispatcher.dispatchPreScroll(
                                available = Offset(x = 0f, y = 0f), source = NestedScrollSource.Drag
                            )

                            // 2. 父组件消费完之后，剩余的滑动距离，自己按需消费

                            // 计算父组件消费后剩余的可使用的滑动距离
                            val availableY = onDelta - consumed.y

                            // canConsumeY 是当前需要消费掉的距离
                            val canConsumeY = if (availableY < 0) { // 向上滑动
                                val dH = minHeightPx - topHeightPx  // 向上滑动过程中，还差多少达到最小高度
                                if (availableY > dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最小高度，就将当前可用距离全部消费掉
                                    availableY
                                } else {  // 如果当前可用的滑动距离足够达到最小高度，就只消费掉需要的距离
                                    dH
                                }
                            } else { // 下滑
                                val dH = maxHeightPx - topHeightPx  // 向下滑动过程中，还差多少达到最大高度
                                if (availableY < dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最大高度，就将当前可用距离全部消费掉
                                    availableY
                                } else {  // 如果当前可用的滑动距离足够达到最大高度，就只消费掉需要的距离
                                    dH
                                }
                            }

                            // 把当前消费掉的距离给到图片高度
                            topHeightPx += canConsumeY

                            // 父组件消费后，以及本次消费后，最后剩余的滑动距离
                            val remain = onDelta - consumed.y - canConsumeY

                            // 3. 自己消费完之后，还有剩余的滑动距离，再给到父组件
                            dispatcher.dispatchPostScroll(
                                consumed = Offset(x = 0f, y = consumed.y + canConsumeY), // 这里是总共消费的滑动距离，包括父组件消费的和本次自己消费的
                                available = Offset(0f, remain),  // 剩余可用的滑动距离
                                source = NestedScrollSource.Drag
                            )
                        }, orientation = Orientation.Vertical
                    )
                    .nestedScroll(connection, dispatcher)
            )
        }
        for (j in 11..40) {
            item {
                Text(text = "item --> $j", modifier = Modifier.fillMaxWidth())
            }
        }
    }
}
```

关键代码都已经加上了注释。看起来应该是非常清晰的。

这里，主要是内部使用 dispatcher 进行事件拦截。

### 3.2 按照分发顺序依次消费
在 3.1 的例子中，头部图片是不检测滑动事件的，手指按住图片滑动是不会响应的，现在需要修改为按住上面图片也是可以滑动，将头部收缩和展开。

下面开始改造:

1. 给 Column 加上 Modifier.draggable 修饰

```
Column(
        modifier = Modifier
            .fillMaxSize()
            .draggable(
                state = rememberDraggableState { onDelta ->

                },
                orientation = Orientation.Vertical
            )
            .nestedScroll(connection = connection)
) {
    ...
}
```

2. 声明 dispatcher，使用 dispatcher 处理嵌套滑动事件

```
val dispatcher = remember { NestedScrollDispatcher() }

Column(
    modifier = Modifier
        .fillMaxSize()
        .draggable(
            state = rememberDraggableState { onDelta ->

                // 1. 滑动距离，给到父组件先消费
                // 调用父组件劫持滑动事件，让父组件先消费，返回值是父组件消费掉的滑动距离
                val consumed = dispatcher.dispatchPreScroll(
                    available = Offset(x = 0f, y = onDelta), source = NestedScrollSource.Drag
                )

                // 2. 父组件消费完之后，剩余的滑动距离，自己按需消费

                // 计算父组件消费后剩余的可使用的滑动距离
                val availableY = (onDelta - consumed.y)

                // consume 是当前需要消费掉的距离
                val consumeY = if (availableY < 0) { // 向上滑动
                    val dH = minHeightPx - topHeightPx  // 向上滑动过程中，还差多少达到最小高度
                    if (availableY > dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最小高度，就将当前可用距离全部消费掉
                        availableY
                    } else {  // 如果当前可用的滑动距离足够达到最小高度，就只消费掉需要的距离
                        dH
                    }
                } else { // 下滑
                    val dH = maxHeightPx - topHeightPx  // 向下滑动过程中，还差多少达到最大高度
                    if (availableY < dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最大高度，就将当前可用距离全部消费掉
                        availableY
                    } else {  // 如果当前可用的滑动距离足够达到最大高度，就只消费掉需要的距离
                        dH
                    }
                }

                // 把当前消费掉的距离给到图片高度
                topHeightPx += consumeY

                // 父组件消费后，以及本次消费后，最后剩余的滑动距离
                val remain = onDelta - consumed.y - consumeY

                // 3. 自己消费完之后，还有剩余的滑动距离，再给到父组件
                dispatcher.dispatchPostScroll(
                    consumed = Offset(x = 0f, y = consumed.y + consumeY), // 这里是总共消费的滑动距离，包括父组件消费的和本次自己消费的
                    available = Offset(0f, remain),  // 剩余可用的滑动距离
                    source = NestedScrollSource.Drag
                )
            },
             orientation = Orientation.Vertical
        )
        .nestedScroll(
            connection = connection, 
            dispatcher = dispatcher
        )
) {
    ...
}
```

同样代码注释已经写得非常清晰了。

完整代码如下：

```
@Composable
fun NestedScrollDemo2() {
    val minHeight = 80.dp
    val maxHeight = 200.dp
    val density = LocalDensity.current

    val minHeightPx = with(density) {
        minHeight.toPx()
    }

    val maxHeightPx = with(density) {
        maxHeight.toPx()
    }

    var topHeightPx by remember {
        mutableStateOf(maxHeightPx)
    }

    val dispatcher = remember { NestedScrollDispatcher() }

    val connection = remember {
        object : NestedScrollConnection {
            /**
             * 预先劫持滑动事件，消费后再交由子布局。
             * available：当前可用的滑动事件偏移量
             * source：滑动事件的类型
             * 返回值：当前组件消费的滑动事件偏移量，如果不想消费可返回Offset.Zero
             */
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                if (source == NestedScrollSource.Drag) {  // 判断是滑动事件
                    if (available.y < 0) { // 向上滑动
                        val dH = minHeightPx - topHeightPx  // 向上滑动过程中，还差多少达到最小高度
                        if (available.y > dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最小高度，就将当前可用距离全部消费掉
                            topHeightPx += available.y
                            return Offset(x = 0f, y = available.y)
                        } else {  // 如果当前可用的滑动距离足够达到最小高度，就只消费掉需要的距离。剩余的给到子组件。
                            topHeightPx += dH
                            return Offset(x = 0f, y = dH)
                        }
                    } else { // 下滑
                        val dH = maxHeightPx - topHeightPx  // 向下滑动过程中，还差多少达到最大高度
                        if (available.y < dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最大高度，就将当前可用距离全部消费掉
                            topHeightPx += available.y
                            return Offset(x = 0f, y = available.y)
                        } else {  // 如果当前可用的滑动距离足够达到最大高度，就只消费掉需要的距离。剩余的给到子组件。
                            topHeightPx += dH
                            return Offset(x = 0f, y = dH)
                        }
                    }
                } else {  // 如果不是滑动事件，就不消费。
                    return Offset.Zero
                }
            }

            /**
             * 获取子布局处理后的滑动事件
             * consumed：之前消费的所有滑动事件偏移量
             * available：当前剩下还可用的滑动事件偏移量
             * source：滑动事件的类型
             * 返回值：当前组件消费的滑动事件偏移量，如果不想消费可返回 Offset.Zero ，则剩下偏移量会继续交由当前布局的父布局进行处理
             */
            override fun onPostScroll(
                consumed: Offset, available: Offset, source: NestedScrollSource
            ): Offset {
                // 子组件处理后的剩余的滑动距离，此处不需要消费了，直接不消费。
                return Offset.Zero
            }

            override suspend fun onPreFling(available: Velocity): Velocity {
                return super.onPreFling(available)
            }

            override suspend fun onPostFling(consumed: Velocity, available: Velocity): Velocity {
                return super.onPostFling(consumed, available)
            }
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .draggable(
                state = rememberDraggableState { onDelta ->

                    // 1. 滑动距离，给到父组件先消费
                    // 调用父组件劫持滑动事件，让父组件先消费，返回值是父组件消费掉的滑动距离
                    val consumed = dispatcher.dispatchPreScroll(
                        available = Offset(x = 0f, y = onDelta), source = NestedScrollSource.Drag
                    )

                    // 2. 父组件消费完之后，剩余的滑动距离，自己按需消费

                    // 计算父组件消费后剩余的可使用的滑动距离
                    val availableY = (onDelta - consumed.y)

                    // consume 是当前需要消费掉的距离
                    val consumeY = if (availableY < 0) { // 向上滑动
                        val dH = minHeightPx - topHeightPx  // 向上滑动过程中，还差多少达到最小高度
                        if (availableY > dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最小高度，就将当前可用距离全部消费掉
                            availableY
                        } else {  // 如果当前可用的滑动距离足够达到最小高度，就只消费掉需要的距离
                            dH
                        }
                    } else { // 下滑
                        val dH = maxHeightPx - topHeightPx  // 向下滑动过程中，还差多少达到最大高度
                        if (availableY < dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最大高度，就将当前可用距离全部消费掉
                            availableY
                        } else {  // 如果当前可用的滑动距离足够达到最大高度，就只消费掉需要的距离
                            dH
                        }
                    }

                    // 把当前消费掉的距离给到图片高度
                    topHeightPx += consumeY

                    // 父组件消费后，以及本次消费后，最后剩余的滑动距离
                    val remain = onDelta - consumed.y - consumeY

                    // 3. 自己消费完之后，还有剩余的滑动距离，再给到父组件
                    dispatcher.dispatchPostScroll(
                        consumed = Offset(x = 0f, y = consumed.y + consumeY), // 这里是总共消费的滑动距离，包括父组件消费的和本次自己消费的
                        available = Offset(0f, remain),  // 剩余可用的滑动距离
                        source = NestedScrollSource.Drag
                    )
                }, orientation = Orientation.Vertical
            )
            .nestedScroll(
                connection = connection, dispatcher = dispatcher
            )
    ) {
        Image(
            painter = painterResource(id = R.mipmap.rc_1),
            contentDescription = null,
            contentScale = ContentScale.FillBounds,
            modifier = Modifier
                .fillMaxWidth()
                .height(with(density) {
                    topHeightPx.toDp()
                })
        )

        LazyColumn {
            repeat(50) {
                item {
                    Text(text = "item --> $it", modifier = Modifier.fillMaxWidth())
                }
            }
        }
    }
}
```

运行效果：

![](/assets/images/技术/编程/Jetpack%20Compose/compose8/pic3.gif)

看效果图不明显，实际上就是按住图片位置拖动是可以收缩和展开顶部图片的。
当然，其实要实现这个效果，也不用整这么复杂，完全可以给 Image 设置 draggable 修饰符来实现：

```
Image(
    painter = painterResource(id = R.mipmap.rc_1),
    contentDescription = null,
    contentScale = ContentScale.FillBounds,
    modifier = Modifier
        .fillMaxWidth()
        .height(with(density) {
            topHeightPx.toDp()
        })
        .draggable(
            state = rememberDraggableState { onDelta ->
                val consumeY = if (onDelta < 0) { // 向上滑动
                    val dH = minHeightPx - topHeightPx  // 向上滑动过程中，还差多少达到最小高度
                    if (onDelta > dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最小高度，就将当前可用距离全部消费掉
                        onDelta
                    } else {  // 如果当前可用的滑动距离足够达到最小高度，就只消费掉需要的距离
                        dH
                    }
                } else { // 下滑
                    val dH = maxHeightPx - topHeightPx  // 向下滑动过程中，还差多少达到最大高度
                    if (onDelta < dH) {  // 如果当前可用的滑动距离全部消费都不足以达到最大高度，就将当前可用距离全部消费掉
                        onDelta
                    } else {  // 如果当前可用的滑动距离足够达到最大高度，就只消费掉需要的距离
                        dH
                    }
                }
                topHeightPx += consumeY
            },
            orientation = Orientation.Vertical
        )
)
```

这样就可以了。

## 小结
1. 本文介绍了 Jetpack Compose 中嵌套滚动的相关知识。
Compose 中嵌套滚动事件的分发思想是，滚动事件会预先交给父组件预先处理，父组件处理消费之后，自己处理剩余滚动距离，自己处理消费完之后，还有剩余，会再交给父组件处理。
2. 一般来说，当子组件检测滚动事件，则需要实现 `NestedScrollConnection` 中的 `onPreScroll` 和 `onPostScroll` 方法。当自己检测滚动事件，则需要使用 `NestedScrollDispatcher` 的相关方法对滚动事件进行分发。
3. 另外还有 Fling 事件，惯性滚动，其分发思想与滚动一致，不同的它的值表示速度。另外惯性滚动过程实现比较复杂，Compose 提供了默认实现，`ScrollableDefaults.flingBehavior()`，感兴趣的朋友可以继续研究。