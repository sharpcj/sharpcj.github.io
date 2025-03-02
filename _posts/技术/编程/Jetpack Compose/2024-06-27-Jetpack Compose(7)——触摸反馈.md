---
layout: post
title: "Jetpack Compose(7)——触摸反馈"
date:  2024-06-27 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

本文介绍 Jetpack Compose 中的手势处理。

官方文档的对 Compose 中的交互做了分类，比如指针输入、键盘输入等。本文主要是介绍指针输入，类比传统 View 体系中的事件分发。

说明：在 Compose 中，手势处理是通过 Modifier 实现的。这里，有人可能要反驳，`Button` 这个可组合项，就是专门用来响应点击事件的，莫慌，接着往下看。

## 一、点按手势
### 1.1 Modifier.clickable

```
fun Modifier.clickable(
    enabled: Boolean = true,
    onClickLabel: String? = null,
    role: Role? = null,
    onClick: () -> Unit
)
```

Clickable 修饰符用来监听组件的点击操作，并且当点击事件发生时会为被点击的组件施加一个波纹涟漪效果动画的蒙层。

Clickable 修饰符使用起来非常简单，在绝大多数场景下我们只需要传入 onClick 回调即可，用于处理点击事件。当然你也可以为 enable 参数设置为一个可变状态，通过状态来动态控制启用点击监听。

```
@Composable
fun ClickDemo() {
  var enableState by remember {
    mutableStateOf<Boolean>(true)
  }
  Box(modifier = Modifier
      .size(200.dp)
      .background(Color.Green)
      .clickable(enabled = enableState) {
        Log.d(TAG, "发生单击操作了～")
      }
  )
}
```

这里可以回答上面的问题，关于 Button 可组合项，我们看下 Button 的源码：

```
@OptIn(ExperimentalMaterialApi::class)
@Composable
fun Button(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    interactionSource: MutableInteractionSource = remember { MutableInteractionSource() },
    elevation: ButtonElevation? = ButtonDefaults.elevation(),
    shape: Shape = MaterialTheme.shapes.small,
    border: BorderStroke? = null,
    colors: ButtonColors = ButtonDefaults.buttonColors(),
    contentPadding: PaddingValues = ButtonDefaults.ContentPadding,
    content: @Composable RowScope.() -> Unit
) {
    val contentColor by colors.contentColor(enabled)
    Surface(
        onClick = onClick,
        modifier = modifier.semantics { role = Role.Button },
        enabled = enabled,
        shape = shape,
        color = colors.backgroundColor(enabled).value,
        contentColor = contentColor.copy(alpha = 1f),
        border = border,
        elevation = elevation?.elevation(enabled, interactionSource)?.value ?: 0.dp,
        interactionSource = interactionSource,
    ) {
        // ... 省略其它代码
    }
}
```

实际是 `surface` 组件响应的 onClick 事件。

```
@ExperimentalMaterialApi
@Composable
fun Surface(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    shape: Shape = RectangleShape,
    color: Color = MaterialTheme.colors.surface,
    contentColor: Color = contentColorFor(color),
    border: BorderStroke? = null,
    elevation: Dp = 0.dp,
    interactionSource: MutableInteractionSource = remember { MutableInteractionSource() },
    content: @Composable () -> Unit
) {
    val absoluteElevation = LocalAbsoluteElevation.current + elevation
    CompositionLocalProvider(
        LocalContentColor provides contentColor,
        LocalAbsoluteElevation provides absoluteElevation
    ) {
        Box(
            modifier = modifier
                .minimumInteractiveComponentSize()
                .surface(
                    shape = shape,
                    backgroundColor = surfaceColorAtElevation(
                        color = color,
                        elevationOverlay = LocalElevationOverlay.current,
                        absoluteElevation = absoluteElevation
                    ),
                    border = border,
                    elevation = elevation
                )
                .clickable(
                    interactionSource = interactionSource,
                    indication = rememberRipple(),
                    enabled = enabled,
                    onClick = onClick
                ),
            propagateMinConstraints = true
        ) {
            content()
        }
    }
}
```

我们看到，surface 的实现又是基于 Box， 最终 Box 是通过 Modifier.clickable 响应点击事件的。

### 1.2 Modifier.combinedClickable

```
fun Modifier.combinedClickable(
  enabled: Boolean = true,
  onClickLabel: String? = null,
  role: Role? = null,
  onLongClickLabel: String? = null,
  onLongClick: (() -> Unit)? = null,
  onDoubleClick: (() -> Unit)? = null,
  onClick: () -> Unit
)
```

除了点击事件，我们经常使用到的还有双击、长按等手势需要响应，Compose 提供了 `Modifier.combinedClickable` 用来响应对于长按点击、双击等复合类点击手势，与 Clickable 修饰符一样，他同样也可以监听单击手势，并且也会为被点击的组件施加一个波纹涟漪效果动画的蒙层。

```
@Composable
fun CombinedClickDemo() {
  var enableState by remember {
    mutableStateOf<Boolean>(true)
  }
  Box(modifier = Modifier
    .size(200.dp)
    .background(Color.Green)
    .combinedClickable(
      enabled = enableState,
      onLongClick = {
        Log.d(TAG, "发生长按点击操作了～")
      },
      onDoubleClick = {
        Log.d(TAG, "发生双击操作了～")
      },
      onClick = {
        Log.d(TAG, "发生单击操作了～")
      }
    )
  )
}
```

## 二、滚动手势
这里所说的滚动，是指可组合项的内容发生滚动，如果想要显示列表，请考虑使用 LazyXXX 系列组件。

### 2.1 滚动修饰符 Modifier.verticalScorll / Modifier.horizontalScorll

```
fun Modifier.verticalScroll(
    state: ScrollState,
    enabled: Boolean = true,
    flingBehavior: FlingBehavior? = null,
    reverseScrolling: Boolean = false
)
```

- state 表示滚动状态
- enabled 表示是否启用 / 禁用该滚动
- flingBehavior 参数表示拖动结束之后的 fling 行为，默认为 null， 会使用 ScrollableDefaults. flingBehavior 策略。
- reverseScrolling， false 表示 ScrollState 为 0 时对应最顶部 top， ture 表示 ScrollState 为 0 时对应底部 bottom。
**注意：**这个反转不是指滚动方向反转，而是对 state 的反转，当 state 为 0 时，即处于列表最底部
大多数场景，我们只需要传入 state 即可。

```
@Composable
fun TestScrollBox() {
    Column(
        modifier = Modifier
            .background(Color.LightGray)
            .wrapContentWidth()
            .height(80.dp)
            .verticalScroll(
                state = rememberScrollState()
            )
    ) {
        repeat(20) {
            Text("item --> $it")
        }
    }
}
```

![](/assets/images/技术/编程/Jetpack%20Compose/compose7/pic1.gif)

借助 ScrollState，您可以更改滚动位置或获取其当前状态。比如滚动到初始位置，则可以调用

```
state.scrollTo(0)
```

对于 `Modifier.horizontalScorll`，从命名可以看出，`Modifier.verticalScorll` 用来实现垂直方向滚动，而 `Modifier.horizontalScorll` 用来实现水平方向的滚动。这里不再赘述了。

```
fun Modifier.horizontalScroll(
    state: ScrollState,
    enabled: Boolean = true,
    flingBehavior: FlingBehavior? = null,
    reverseScrolling: Boolean = false
)
```

### 2.2 可滚动修饰符 Modifier.scrollable
`scrollable` 修饰符与滚动修饰符不同，scrollable 会检测滚动手势并捕获增量，但不会自动偏移其内容。而是通过 `ScrollableState` 委派给用户，而这是此修饰符正常运行所必需的。

构建 `ScrollableState` 时，您必须提供一个 `consumeScrollDelta` 函数，该函数将在每个滚动步骤（通过手势输入、平滑滚动或快速滑动）调用，并且增量以像素为单位。此函数必须返回使用的滚动距离量，以确保在存在具有 `scrollable` 修饰符的嵌套元素时正确传播事件。

>注意：scrollable 修饰符不会影响应用该修饰符的元素的布局。这意味着，对元素布局或其子项所做的任何更改都必须通过 ScrollableState 提供的增量进行处理。另外还需要注意的是，scrollable 并不考虑子项的布局，这意味着它不需要测量子项即可传播滚动增量。

```
@Composable
fun rememberScrollableState(consumeScrollDelta: (Float) -> Float): ScrollableState {
    val lambdaState = rememberUpdatedState(consumeScrollDelta)
    return remember { ScrollableState { lambdaState.value.invoke(it) } }
}
```

看个具体例子：

```
@Composable
fun TestScrollableBox() {
    var offsetY by remember {
        mutableFloatStateOf(0f)
    }
    Column(modifier = Modifier
        .background(Color.LightGray)
        .wrapContentWidth()
        .offset(
            y = with(LocalDensity.current) {
                offsetY.toDp()
            }
        )
        .scrollable(
            orientation = Orientation.Vertical,
            state = rememberScrollableState { consumeScrollDelta ->
                offsetY += consumeScrollDelta
                consumeScrollDelta
            }
        )) {
        repeat(20) {
            Text("item --> $it")
        }
    }
}
```

运行效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose7/pic2.gif)

其实很好理解，`rememberScrollableState` 提供了滚动的偏移量，需要自己对偏移量进行处理，并且需要指定消费。

除了自己实现 rememberScrollableState 之外，也可以用前面的 `rememberScrollState`，它提供了一个默认的实现，将滚动数据存储在 ScrollState 的 value 中，并消费掉所有的滚动距离。但是 ScrollState 的值的范围是大于 0 的，无法出现负数。

```
@Stable
class ScrollState(initial: Int) : ScrollableState {

}
```

可以看到 `ScrollState` 实现了 `ScrollableState` 这个接口。

另外，`Modifier.scrollable` 可滚动修饰符需要指定滚动方向垂直或者水平。相对而言，该修饰符处于更低级别，灵活性更强，而上一小节讲到的滚动修饰符则是基于 `Modifier.scrollable` 实现的。理解好两者的区别，才能在实际开发中选择合适的 API。

## 三、 拖动手势
### 3.1 Modifier.draggable
Modifier.draggable 修饰符只能监听水平或者垂直方向的拖动偏移。

```
fun Modifier.draggable(
    state: DraggableState,
    orientation: Orientation,
    enabled: Boolean = true,
    interactionSource: MutableInteractionSource? = null,
    startDragImmediately: Boolean = false,
    onDragStarted: suspend CoroutineScope.(startedPosition: Offset) -> Unit = {},
    onDragStopped: suspend CoroutineScope.(velocity: Float) -> Unit = {},
    reverseDirection: Boolean = false
)
```

最主要的参数 `state` 需要可以记录拖动状态，获取到拖动手势的偏移量。`orientation` 指定监听拖动的方向。
使用示例：

```
@Composable
fun DraggableBox() {
    var offsetX by remember {
        mutableFloatStateOf(0f)
    }
    Box(modifier = Modifier
        .offset {
            IntOffset(x = offsetX.roundToInt(), y = 0)
        }
        .background(Color.LightGray)
        .size(80.dp)
        .draggable(
            orientation = Orientation.Horizontal,
            state = rememberDraggableState { onDelta ->
                offsetX += onDelta
            }
        ))
}
```

![](/assets/images/技术/编程/Jetpack%20Compose/compose7/pic3.gif)

注意：拖动手势本身不会让 UI 发生变化。通过 rememberDraggableState 构造一个 DraggableState，获取拖动偏移量，然后把这个偏移量累加到某个状态变量上，利用这个状态来改变 UI 界面。
比如这里使用了 offset 去改变组件的偏移量。

**注意：由于Modifer链式执行，此时offset必需在draggable与background前面。**

### 3.2 Modifier.draggable2D
`Modifier.draggable` 通过 `orientation` 参数指定方向，只能水平或者垂直方向拖动。而 `Modifier.draggable2D` 则是可以同时沿着水平或者垂直方向拖动。用法如下：

```
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun Draggable2DBox() {
    var offset by remember {
        mutableStateOf(Offset.Zero)
    }
    Box(modifier = Modifier
        .offset {
            IntOffset(x = offset.x.roundToInt(), y = offset.y.roundToInt())
        }
        .background(Color.LightGray)
        .size(80.dp)
        .draggable2D(
            state = rememberDraggable2DState { onDelta ->
                offset += onDelta
            }
        ))
}
```

与 `Modifier.draggable` 相比，`Modifier.draggable2D` 修饰符没有了 `orientation` 参数，无需指定方向，同时，state 类型是 Draggable2DState。构造该 State 的 lambda 表达式的参数，delta 类型也变成了 Offset 类型。这样就实现了在 2D 平面上的任意方向拖动。

![](/assets/images/技术/编程/Jetpack%20Compose/compose7/pic4.gif)

## 四、锚定拖动
Modifier.anchoredDraggable 是 Jetpack Compose 1.6.0 引入的一个新的修饰符，替代了 Swipeable, 用来实现锚定拖动。

```
fun <T> Modifier.anchoredDraggable(
    state: AnchoredDraggableState<T>,
    orientation: Orientation,
    enabled: Boolean = true,
    reverseDirection: Boolean = false,
    interactionSource: MutableInteractionSource? = null
)
```

此修饰符参数：

- state 一个 DraggableState 的实例。
- orientation 我们要将内容拖入的方向，水平或者垂直。
- enabled 用于启用/禁用拖动手势。
- reverseDirection 是否反转拖动方向。
- interactionSource 用于拖动手势的可选交互源。

锚定拖动对象有 2 个主要部分，一个是应用于要拖动的内容的修饰符，另一个是其状态 AnchoredDraggableState，它指定拖动的操作方式。

除了构造函数之外，为了使用 anchoredDraggable 修饰符，我们还需要熟悉其他几个 API，它们是 updateAnchors 和 requireOffset。

### 4.1 可拖动状态 AnchoredDraggableState

```
class AnchoredDraggableState<T>(
    initialValue: T,
    internal val positionalThreshold: (totalDistance: Float) -> Float,
    internal val velocityThreshold: () -> Float,
    val animationSpec: AnimationSpec<Float>,
    internal val confirmValueChange: (newValue: T) -> Boolean = { true }
)
```
在这个构造函数中，我们有

- initialValue，一个参数化参数，用于在首次呈现时捕捉可拖动的内容。
- positionalThreshold 一个 lambda 表达式，用于根据锚点之间的距离确定内容是以动画形式呈现到下一个锚点还是返回到原始锚点。
- velocityTheshold 一个 lambda 表达式，它返回一个速度，用于确定我们是否应该对下一个锚点进行动画处理，而不考虑位置Theshold。如果拖动速度超过此阈值，那么我们将对下一个锚点进行动画处理，否则使用 positionalThreshold。
- animationSpec，用于确定如何对可拖动内容进行动画处理。
- confirmValueChange 一个lambda 表达式，可选参数，可用于否决对可拖动内容的更改。

值得注意的是，目前没有可用的 rememberDraggableState 工厂方法，因此我们需要通过 remember 手动定义可组合文件中的状态。

### 4.2 updateAnchors

```
fun updateAnchors(
    newAnchors: DraggableAnchors<T>,
    newTarget: T = if (!offset.isNaN()) {
        newAnchors.closestAnchor(offset) ?: targetValue
    } else targetValue
)
```

我们使用 updateAnchors 方法指定内容将捕捉到的拖动区域上的停止点。我们至少需要指定 2 个锚点，以便可以在这 2 个锚点之间拖动内容，但我们可以根据需要添加任意数量的锚点。

### 4.3 requireOffset
此方法仅返回可拖动内容的偏移量，以便我们可以将其应用于内容。同样，anchoredDraggable 修饰符本身不会在拖动时移动内容，它只是计算用户在屏幕上拖动时的偏移量，我们需要自己根据 requireOffset 提供的偏移量更新内容。

### 4.4 使用示例介绍

```
// 1. 定义锚点
enum class DragAnchors {
    Start,
    End,
}

@OptIn(ExperimentalFoundationApi::class)
@Composable
fun DragAnchorDemo() {
    val density = LocalDensity.current

    // 2. 使用 remember 声明 AnchoredDraggableState,确保重组过程中能够缓存结果
    val state = remember {
        AnchoredDraggableState(
            // 3. 设置 AnchoredDraggableState 的初始锚点值
            initialValue = DragAnchors.Start,

            // 4. 根据行进的距离确定我们是否对下一个锚点进行动画处理。在这里，我们指定确定我们是否移动到下一个锚点的阈值是到下一个锚点距离的一半——如果我们移动了两个锚点之间的半点，我们将对下一个锚点进行动画处理，否则我们将返回到原点锚点。
            positionalThreshold = { totalDistance ->
                totalDistance * 0.5f
            },
            
            // 5.确定将触发拖动内容以动画形式移动到下一个锚点的最小速度，而不管是否 已达到 positionalThreshold 指定的阈值。
            velocityThreshold = {
                with(density) {
                    100.dp.toPx()
                }
            },

            // 6. 指定了在释放拖动手势时如何对下一个锚点进行动画处理;这里我们使用 一个补间动画，它默认为 FastOutSlowIn 插值器
            animationSpec = tween(),

            confirmValueChange = { newValue ->
                true
            }
        ).apply {

            // 7. 使用前面介绍的 updateAnchors 方法定义内容的锚点 
            updateAnchors(

                // 8. 使用 DraggableAnchors 帮助程序方法指定要使用的锚点 。我们在这里所做的是创建 DragAnchors 到内容的实际偏移位置的映射。在这种情况下，当状态为“开始”时，内容将偏移量为 0 像素，当状态为“结束”时，内容偏移量将为 400 像素。
                DraggableAnchors {
                    DragAnchors.Start at 0f
                    DragAnchors.End at 800f
                }
            )
        }
    }

    Box {
        Image(
            painter = painterResource(id = R.mipmap.ic_test), contentDescription = null,
            modifier = Modifier
                .offset {
                    IntOffset(x = 0, y = state.requireOffset().roundToInt())
                }
                .clip(CircleShape)
                .size(80.dp)
                // 使用前面定义的状态
                .anchoredDraggable(
                    state = state,
                    orientation = Orientation.Vertical
                )
        )
    }
}
```

**注意： offset 要先于 anchoredDraggable 调用**

看看效果:

![](/assets/images/技术/编程/Jetpack%20Compose/compose7/pic5.gif)

拖动超多一半的距离，或者速度超过阈值，就会以动画形式跳到下一个锚点。

## 五、转换手势
### 5.1 Modifier.transformer
Modifier.transformer 修饰符允许开发者监听 UI 组件的双指拖动、缩放或旋转手势，通过所提供的信息来实现 UI 动画效果。

```
@ExperimentalFoundationApi
fun Modifier.transformable(
    state: TransformableState,
    canPan: (Offset) -> Boolean,
    lockRotationOnZoomPan: Boolean = false,
    enabled: Boolean = true
)
```

- transformableState 必传参数，可以使用 rememberTransformableState 创建一个 transformableState, 通过 rememberTransformableState 的尾部 lambda 可以获取当前双指拖动、缩放或旋转手势信息。

- lockRotationOnZoomPan 可选参数，当主动设置为 true 时，当UI组件已发生双指拖动或缩放时，将获取不到旋转角度偏移量信息。

使用示例：

```
@Composable
fun TransformBox() {
    var offset by remember { mutableStateOf(Offset.Zero) }
    var rotationAngle by remember { mutableStateOf(0f) }
    var scale by remember { mutableStateOf(1f) }

    Box(modifier = Modifier
        .size(80.dp)
        .rotate(rotationAngle) // 需要注意 offset 与 rotate 的调用先后顺序
        .offset {
            IntOffset(offset.x.roundToInt(), offset.y.roundToInt())
        }
        .scale(scale)
        .background(Color.LightGray)
        .transformable(
            state = rememberTransformableState { zoomChange: Float, panChange: Offset, rotationChange: Float ->
                scale *= zoomChange
                offset += panChange
                rotationAngle += rotationChange
            }
        )
    )
}
```

注意：由于 Modifer 链式执行，此时需要注意 offset 与 rotate 调用的先后顺序
⚠️示例( offset 在 rotate 前面): 一般情况下我们都需要组件在旋转后，当出现双指拖动时组件会跟随手指发生偏移。若 offset 在 rotate 之前调用，则会出现组件旋转后，当双指拖动时组件会以当前旋转角度为基本坐标轴进行偏移。这是由于当你先进行 offset 说明已经发生了偏移，而 rotate 时会改变当前UI组件整个坐标轴，所以出现与预期不符的情况出现。

效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose7/pic6.gif)

## 六、自定义触摸反馈
### 6.1 Modifier.pointerInput
前面已经介绍完常用的手势处理了，都非常简单。但是有时候我们需要自定义触摸反馈。这时候可以就需要使用到 `Modifier.PointerInput` 修饰符了。该修饰符提供了更加底层细粒度的手势检测，前面讲到的高级别修饰符实际上最终都是用底层低级别 API 来实现的。

```
fun Modifier.pointerInput(
    vararg keys: Any?,
    block: suspend PointerInputScope.() -> Unit
)
```

看下参数：

- keys 当 Composable 组件发生重组时，如果传入的 keys 发生了变化，则手势事件处理过程会被中断。
- block 在这个 PointerInputScope 类型作用域代码块中我们便可以声明手势事件处理逻辑了。通过 suspend 关键字可知这是个协程体，这意味着在 Compose 中手势处理最终都发生在协程中。

在 PointerInputScope 作用域内，可以使用更加底层的手势检测的基础API。

#### 6.1.1 点击类型的基础 API
|API名称|作用|
|---|---|
|detectTapGestures|监听点击手势|

```
suspend fun PointerInputScope.detectTapGestures(
  onDoubleTap: ((Offset) -> Unit)? = null,
  onLongPress: ((Offset) -> Unit)? = null,
  onPress: suspend PressGestureScope.(Offset) -> Unit = NoPressGesture,
  onTap: ((Offset) -> Unit)? = null
)
```

看一下这几个方法名，就能知道方法的作用。使用起来与前面讲解高级的修饰符差不多。在 `PointerInputScope` 中使用 `detectTapGestures`，不会带有涟波纹效果，方便我们根据需要进行定制。

- onDoubleTap (可选)：双击时回调
- onLongPress (可选)：长按时回调
- onPress (可选)：按下时回调
- onTap (可选)：轻触时回调

>这几种点击事件回调存在着先后次序的，并不是每次只会执行其中一个。onPress 是最普通的 ACTION_DOWN 事件，你的手指一旦按下便会回调。如果你连着按了两下，则会在执行两次 onPress 后执行 onDoubleTap。如果你的手指按下后不抬起，当达到长按的判定阈值 (400ms) 会执行 onLongPress。如果你的手指按下后快速抬起，在轻触的判定阈值内(100ms)会执行 onTap 回调。

总的来说， **onDoubleTap 回调前必定会先回调 2 次 Press，而 onLongPress 与 onTap 回调前必定会回调 1 次 Press**。

使用如下：

```
@Composable
fun PointerInputDemo() {
    Box(modifier = Modifier.background(Color.LightGray).size(100.dp)
        .pointerInput(Unit) {
            detectTapGestures(
                onDoubleTap = {
                    Log.i("sharpcj", "onDoubleTap --> $it")
                },
                onLongPress = {
                    Log.i("sharpcj", "onLongPress --> $it")
                },
                onPress = {
                    Log.i("sharpcj", "onPress --> $it")
                },
                onTap = {
                    Log.i("sharpcj", "onTap --> $it")
                }
            )
        }
    )
}
```
    
#### 6.1.2 拖动类型基础 API
|API名称|作用|
|---|---|
|detectDragGestures|监听拖动手势|
|detectDragGesturesAfterLongPress|监听长按后的拖动手势|
|detectHorizontalDragGestures|监听水平拖动手势|
|detectVerticalDragGestures|监听垂直拖动手势|

以 `detectDragGesturesAfterLongPress` 为例：

```
@Composable
fun PointerInputDemo() {
    Box(modifier = Modifier.background(Color.LightGray).size(100.dp)
        .pointerInput(Unit) {
            detectDragGesturesAfterLongPress(
                onDragStart = {

                },
                onDrag = { change: PointerInputChange, dragAmount: Offset ->

                },
                onDragEnd = {

                },
                onDragCancel = {

                }
            )
        }
    )
}
```

该 API 会检测长按后的拖动，提供了四个回调时机，onDragStart 会在拖动开始时回调，onDragEnd 会在拖动结束时回调，onDragCancel 会在拖动取消时回调，而 onDrag 则会在拖动真正发生时回调。

**注意:**

>1. onDragCancel 触发时机多发生于滑动冲突的场景，子组件可能最开始是可以获取到拖动事件的，当拖动手势事件达到莫个指定条件时可能会被父组件劫持消费，这种场景下便会执行 onDragCancel 回调。所以 onDragCancel 回调主要依赖于实际业务逻辑。
>2. 上述 API 会检测长按后的拖动，但是其本身并没有提供长按时的回调方法。如果要同时监听长按，可以配合 detectTapGestures 一起使用。
由于这些检测器是顶级检测器，因此无法在一个 pointerInput 修饰符中添加多个检测器。以下代码段只会检测点按操作，而不会检测拖动操作。

```
var log by remember { mutableStateOf("") }
Column {
    Text(log)
    Box(
        Modifier
            .size(100.dp)
            .background(Color.Red)
            .pointerInput(Unit) {
                detectTapGestures { log = "Tap!" }
                // Never reached
                detectDragGestures { _, _ -> log = "Dragging" }
            }
    )
}
```

在内部，detectTapGestures 方法会阻塞协程，并且永远不会到达第二个检测器。如果需要向可组合项添加多个手势监听器，可以改用单独的 pointerInput 修饰符实例：

```
var log by remember { mutableStateOf("") }
Column {
    Text(log)
    Box(
        Modifier
            .size(100.dp)
            .background(Color.Red)
            .pointerInput(Unit) {
                detectTapGestures { log = "Tap!" }
            }
            .pointerInput(Unit) {
                // These drag events will correctly be triggered
                detectDragGestures { _, _ -> log = "Dragging" }
            }
    )
}
```

#### 6.1.3 转换类型基础 API
|API名称|作用|
|---|---|
|detectTransformGestures|监听拖动、缩放与旋转手势|

```
@Composable
fun PointerInputDemo() {
    Box(modifier = Modifier.background(Color.LightGray).size(100.dp)
        .pointerInput(Unit) {
                detectTransformGestures(
                    panZoomLock = false,
                    onGesture = { centroid: Offset, pan: Offset, zoom: Float, rotation: Float ->

                    }
                )
            }
        )
}
```

与 `Modifier.transfomer` 修饰符不同的是，通过这个 API 可以监听单指的拖动手势，和拖动类型基础API所提供的功能一样，除此之外还支持监听双指缩放与旋转手势。反观 `Modifier.transfomer` 修饰符只能监听到双指拖动手势。

- panZoomLock(可选)： 当拖动或缩放手势发生时是否支持旋转
- onGesture(必须)：当拖动、缩放或旋转手势发生时回调

### 6.2 awaitPointerEventScope
前面介绍的 GestureDetector 系列 API 本质上仍然是一种封装，既然手势处理是在协程中完成的，所以手势监听必然是通过协程的挂起恢复实现的，以取代传统的回调监听方式。

在 `PointerInputScope` 中我们使用 `awaitPointerEventScope` 方法获得 `AwaitPointerEventScope` 作用域，在 `AwaitPointerEventScope` 作用域中我们可以使用 Compose 中所有低级别的手势处理挂起方法。当 `awaitPointerEventScope` 内所有手势事件都处理完成后 `awaitPointerEventScope` 便会恢复执行将 Lambda 中最后一行表达式的数值作为返回值返回。

```
suspend fun <R> awaitPointerEventScope(
    block: suspend AwaitPointerEventScope.() -> R
): R
```

在 `AwaitPointerEventScope` 中提供了一些基础手势方法：

|API名称|作用|
|---|---|
|awaitPointerEvent|手势事件|
|awaitFirstDown|第一根手指的按下事件|
|drag|拖动事件|
|horizontalDrag|水平拖动事件|
|verticalDrag|垂直拖动事件|
|awaitDragOrCancellation|单次拖动事件|
|awaitHorizontalDragOrCancellation|单次水平拖动事件|
|awaitVerticalDragOrCancellation|单次垂直拖动事件|
|awaitTouchSlopOrCancellation|有效拖动事件|
|awaitHorizontalTouchSlopOrCancellation|有效水平拖动事件|
|awaitVerticalTouchSlopOrCancellation|有效垂直拖动事件|

#### 6.2.1 原始时间 awaitPointerEvent
上层所有手势监听 API 都是基于这个 API 实现的，他的作用类似于传统 View 中的 onTouchEvent() 。无论用户是按下、移动或抬起都将视作一次手势事件，当手势事件发生时 awaitPointerEvent 便会恢复返回监听到的屏幕上所有手指的交互信息。

以下代码可以用来监听原始的指针事件。

```
@Composable
fun PointerEventDemo() {
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(100.dp)
        .pointerInput(Unit) {
            awaitPointerEventScope {
                while (true) {
                    val event = awaitPointerEvent()
                    Log.d(
                        "sharpcj",
                        "event --> type: ${event.type} - x: ${event.changes[0].position.x} - y: ${event.changes[0].position.y}"
                    )
                }
            }
        })
}
```

我们点击，看到日志如下：

```
D  event --> type: Press - x: 188.0 - y: 124.0
D  event --> type: Release - x: 188.0 - y: 124.0
```

我们可以看到事件的 type 为 `Press` 和 `Release`

点击移动，日志如下：

```
D  event --> type: Press - x: 178.0 - y: 178.0
D  event --> type: Move - x: 181.93164 - y: 175.06836
D  event --> type: Move - x: 183.99316 - y: 174.0
D  event --> type: Move - x: 185.5 - y: 171.0
D  event --> type: Move - x: 191.0 - y: 164.0
D  event --> type: Release - x: 191.0 - y: 164.0
```

注意事件的 type 为 `Press`、`Move`、`Release`。

- awaitPointerEventScope 创建可用于等待指针事件的协程作用域
- awaitPointerEvent 会挂起协程，直到发生下一个指针事件

以上监听原始输入事件非常强大，类似于传统 View 中完全实现 onTouchEvent 方法。但是也很复杂。实际场景中几乎不会使用，而是直接使用前面讲到的手势检测 GestureDetect API。

### 6.3 awaitEachGesture
Compose 手势操作实际上是在协程中监听处理的，当协程处理完一轮手势交互后便会结束，当进行第二次手势交互时由于负责手势监听的协程已经结束，手势事件便会被丢弃掉。为了让手势监听协程能够不断地处理每一轮的手势交互，很容易想到可以在外层嵌套一个 while(true) 进行实现，然而这么做并不优雅，且也存在着一些问题。更好的处理方式是使用 awaitEachGesture, awaitEachGesture 方法保证了每一轮手势处理逻辑的一致性。实际上前面所介绍的 GestureDetect 系列 API，其内部实现都使用了 forEachGesture。

```
suspend fun PointerInputScope.awaitEachGesture(block: suspend AwaitPointerEventScope.() -> Unit) {
    val currentContext = currentCoroutineContext()
    awaitPointerEventScope {
        while (currentContext.isActive) {
            try {
                block()

                // Wait for all pointers to be up. Gestures start when a finger goes down.
                awaitAllPointersUp()
            } catch (e: CancellationException) {
                if (currentContext.isActive) {
                    // The current gesture was canceled. Wait for all fingers to be "up" before
                    // looping again.
                    awaitAllPointersUp()
                } else {
                    // detectGesture was cancelled externally. Rethrow the cancellation exception to
                    // propagate it upwards.
                    throw e
                }
            }
        }
    }
}
```

在 awaitEachGesture 中使用特定的手势事件

```
@Composable
fun PointerEventDemo() {
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(100.dp)
        .pointerInput(Unit) {
            awaitEachGesture {
                awaitFirstDown().also { 
                    it.consume()
                    Log.d("sharpcj", "down")
                }
                val up = waitForUpOrCancellation()
                if (up != null) {
                    up.consume()
                    Log.d("sharpcj", "up")
                }
            }
        }
    )
}
```

### 6.3 多指事件
`awaitPointerEvent` 返回一个 `PointerEvent`, PointerEvent 中包含一个集合 `val changes: List<PointerInputChange>` 这里面包含了所有手指的事件信息，我们看看 `PointerInputChange`

```
@Immutable
class PointerInputChange(
    val id: PointerId,
    val uptimeMillis: Long,
    val position: Offset,
    val pressed: Boolean,
    val pressure: Float,
    val previousUptimeMillis: Long,
    val previousPosition: Offset,
    val previousPressed: Boolean,
    isInitiallyConsumed: Boolean,
    val type: PointerType = PointerType.Touch,
    val scrollDelta: Offset = Offset.Zero
) 
```

PointerInputChange 包含某个手指的事件具体信息。
比如前面打印日志的时候，使用了 `event.changes[0].position` 获取坐标信息。

### 6.4 事件分发
#### 6.4.1 事件调度
并非所有指针事件都会发送到每个 pointerInput 修饰符。事件分派的工作原理如下：

- 系统会将指针事件分派给可组合层次结构。新指针触发其第一个指针事件时，系统会开始对“符合条件”的可组合项进行命中测试。如果可组合项具有指针输入处理功能，则会被视为符合条件。命中测试从界面树顶部流向底部。当指针事件发生在可组合项的边界内时，即被视为“命中”。此过程会产生一个“命中测试正例”的可组合项链。
- 默认情况下，当树的同一级别上有多个符合条件的可组合项时，只有 Z-index 最高的可组合项才是“hit”。例如，当您向 Box 添加两个重叠的 Button 可组合项时，只有顶部绘制的可组合项才会收到任何指针事件。从理论上讲，您可以通过创建自己的 PointerInputModifierNode 实现并将 sharePointerInputWithSiblings 设为 true 来替换此行为。
- 系统会将同一指针的其他事件分派到同一可组合项链，并根据事件传播逻辑流动。系统不再对此指针执行命中测试。这意味着链中的每个可组合项都会接收该指针的所有事件，即使这些事件发生在该可组合项的边界之外时。不在链中的可组合项永远不会收到指针事件，即使指针位于其边界内也是如此。
由鼠标或触控笔悬停时触发的悬停事件不属于此处定义的规则。悬停事件会发送给用户点击的任意可组合项。因此，当用户将指针从一个可组合项的边界悬停在下一个可组合项的边界上时，事件会发送到新的可组合项，而不是将事件发送到第一个可组合项。

官方文档的描述比较清楚，为了更加直观，还是自己写示例说明一下:

```
@Composable
fun EventConsumeDemo() {
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(300.dp)
        .pointerInput(Unit) {
            awaitEachGesture {
                while (true) {
                    val event = awaitPointerEvent()
                    Log.d("sharpcj", "outer box --> ${event.type}")
                }
            }
        }) {
        Box(modifier = Modifier
            .background(Color.Yellow)
            .size(200.dp)
            .pointerInput(Unit) {
                awaitEachGesture {
                    while (true) {
                        val event = awaitPointerEvent()
                        Log.d("sharpcj", "inner box --> ${event.type}")
                    }
                }
            })
    }
}
```

如上代码，我们在 inner Box 中轻画一下。日志如下：

```
D  inner box --> Press
D  out box --> Press
D  inner box --> Move
D  out box --> Move
D  inner box --> Move
D  out box --> Move
D  inner box --> Release
D  out box --> Release
```

**解释：**

1. inner Box 和 outer Box 都会收到事件, 因为点击的位置同时处在 inner Box 和 outer Box 之中，
2. 由于 inner Box 的 Z-index 更高，所以先收到事件。

#### 6.4.2 事件消耗
如果为多个可组合项分配了手势处理程序，这些处理程序不应冲突。例如，我们来看看以下界面：

![](/assets/images/技术/编程/Jetpack%20Compose/compose7/pic7.png)

当用户点按书签按钮时，该按钮的 onClick lambda 会处理该手势。当用户点按列表项的任何其他部分时，ListItem 会处理该手势并转到文章。就指针输入而言，Button 必须“消费”此事件，以便其父级知道不会再对其做出响应。开箱组件中包含的手势和常见的手势修饰符就包含这种使用行为，但如果您要编写自己的自定义手势，则必须手动使用事件。可以使用 `PointerInputChange.consume` 方法执行此操作：

```
Modifier.pointerInput(Unit) {

    awaitEachGesture {
        while (true) {
            val event = awaitPointerEvent()
            // consume all changes
            event.changes.forEach { it.consume() }
        }
    }
}
```

使用事件不会阻止事件传播到其他可组合项。可组合项需要明确忽略已使用的事件。编写自定义手势时，您应检查某个事件是否已被其他元素使用：

```
Modifier.pointerInput(Unit) {
    awaitEachGesture {
        while (true) {
            val event = awaitPointerEvent()
            if (event.changes.any { it.isConsumed }) {
                // A pointer is consumed by another gesture handler
            } else {
                // Handle unconsumed event
            }
        }
    }
}
```

看实际场景，当有两个组件叠加起来的时候，我们更多时候只是希望外层的组件响应事件。怎么处理，还是看上面的例子，我们只希望 inner Box 处理事件。修改代码如下：

```
@Composable
fun EventConsumeDemo() {
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(300.dp)
        .pointerInput(Unit) {
            awaitEachGesture {
                while (true) {
                    val event = awaitPointerEvent()
                    if (event.changes.any{ it.isConsumed }) {
                        Log.d("sharpcj", "A pointer is consumed by another gesture handler")
                    } else {
                        Log.d("sharpcj", "out box --> ${event.type}")
                    }
                }
            }
        }) {
        Box(modifier = Modifier
            .background(Color.Yellow)
            .size(200.dp)
            .pointerInput(Unit) {
                awaitEachGesture {
                    while (true) {
                        val event = awaitPointerEvent()
                        Log.d("sharpcj", "inner box --> ${event.type}")
                        event.changes.forEach{
                            it.consume()
                        }
                    }
                }
            })
    }
}
```

结果：

```
D  inner box --> Press
D  A pointer is consumed by another gesture handler
D  inner box --> Move
D  A pointer is consumed by another gesture handler
D  inner box --> Move
D  A pointer is consumed by another gesture handler
D  inner box --> Move
D  A pointer is consumed by another gesture handler
D  inner box --> Release
D  A pointer is consumed by another gesture handler
```

**解释：**

1. 我们在 inner Box 先收到事件并且处理之后，调用 event.changes.forEach { it.consume() } 将所有的事件都消费掉。
2. inner Box 将事件消费掉，并不能阻止 outer Box 收到事件。
3. 需要在 outer Box 中通过判断事件是否被消费，来编写正确的逻辑处理。

再修改下代码，我们使用上层的 GestureDetect API ，再次测试：

```
@Composable
fun EventConsumeDemo2() {
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(300.dp)
        .pointerInput(Unit) {
            detectTapGestures(
                onTap = {
                    Log.d("sharpcj", "outer Box onTap")
                }
            )
        }
    ) {
        Box(modifier = Modifier
            .background(Color.Yellow)
            .size(200.dp)
            .pointerInput(Unit) {
                detectTapGestures(
                    onTap = {
                        Log.d("sharpcj", "inner Box onTap")
                    }
                )
            })
    }
}
```

结果如下：

```
D  inner Box onTap
D  inner Box onTap
D  inner Box onTap
D  inner Box onTap
```

**解释：**
Jetpack Compose 提供的开箱组件中包含的手势和常见的手势修饰符默认就做了上述判断，事件只能被 Z-Index 最高的组件处理。

#### 6.4.3 事件传播
如前所述，指针事件会传递到其命中的每个可组合项。当有多个可组合项“叠”在一起的时候，事件会按什么顺序传播呢？
实际上，事件会有三次流经可组合项：

- Initial 在初始传递中，事件从界面树顶部流向底部。此流程允许父项在子项使用事件之前拦截事件。
- Main 在主传递中，事件从界面树的叶节点一直流向界面树的根。此阶段是您通常使用手势的位置，也是监听事件时的默认传递。处理此传递中的手势意味着叶节点优先于其父节点，这是大多数手势最符合逻辑的行为。在此示例中，Button 会在 ListItem 之前收到事件。
- Final 在“最终通过”中，事件会再一次从界面树顶部流向叶节点。此流程允许堆栈中较高位置的元素响应其父项的事件消耗。例如，当按下按钮变为可滚动父项的拖动时，按钮会移除其涟漪指示。

实际上在诸如 `awaitPointerEvent` 的方法中，有一个参数 PointerEventPass,用来控制事件传播的。

```
suspend fun awaitPointerEvent(
    pass: PointerEventPass = PointerEventPass.Main
)
```

分发顺序：

```
PointerEventPass.Initial -> PointerEventPass.Main -> PointerEventPass.Final
```

对应了上面的描述。
看示例：

```
@Composable
fun EventConsumeDemo() {
    Box(modifier = Modifier
        .background(Color.LightGray)
        .size(300.dp)
        .pointerInput(Unit) {
            awaitEachGesture {
                while (true) {
                    val event = awaitPointerEvent(PointerEventPass.Main)
                    Log.d("sharpcj", "box1 --> ${event.type}")
                }
            }
        }) {
        Box(modifier = Modifier
            .background(Color.Yellow)
            .size(250.dp)
            .pointerInput(Unit) {
                awaitEachGesture {
                    while (true) {
                        val event = awaitPointerEvent(PointerEventPass.Initial)
                        Log.d("sharpcj", "box2 --> ${event.type}")
                    }
                }
            }) {
            Box(modifier = Modifier
                .background(Color.Blue)
                .size(200.dp)
                .pointerInput(Unit) {
                    awaitEachGesture {
                        while (true) {
                            val event = awaitPointerEvent(PointerEventPass.Final)
                            Log.d("sharpcj", "box3 --> ${event.type}")
                        }
                    }
                }) {
                Box(modifier = Modifier
                    .background(Color.Red)
                    .size(150.dp)
                    .pointerInput(Unit) {
                        awaitEachGesture {
                            while (true) {
                                val event = awaitPointerEvent()
                                Log.d("sharpcj", "box4 --> ${event.type}")
                            }
                        }
                    })
            }
        }
    }
}
```

运行结果：

```
D  box2 --> Press
D  box4 --> Press
D  box1 --> Press
D  box3 --> Press
D  box2 --> Move
D  box4 --> Move
D  box1 --> Move
D  box3 --> Move
D  box2 --> Move
D  box4 --> Move
D  box1 --> Move
D  box3 --> Move
D  box2 --> Release
D  box4 --> Release
D  box1 --> Release
D  box3 --> Release
```

**解释：**

1. Initial 传递由根节点到叶子结点依次传递，其中 Box2 拦截了。所有 Box2 优先处理事件。
2. Main 传递由叶子节点传递到父节点， Box1 显示声明了 `PointerEventPass.Main` 和 Box4 没有声明，但是默认参数也是 `PointerEventPass.Main`， 由于是从叶子结点向根节点传播，所以 Box4 先收到事件，然后是 Box1 收到事件。
3. Final 事件再次从根节点传递到叶子结点，这里只有 Box3 参数是 `PointerEventPass.Final`，所以 Box3 最后收到事件。

以上是事件传播的分析，关于消费，同理，如果先收到事件的可组合项把事件消费了，后收到事件的组件根据需要判断事件是否被消费即可。

## 七、嵌套滚动 Modifier.NestedScroll
关于嵌套滚动，相对复杂一点。不过在 Compose 中，使用 `Modifier.NestedScroll` 修饰符来实现，也不难学。
下一篇文章单独来介绍。