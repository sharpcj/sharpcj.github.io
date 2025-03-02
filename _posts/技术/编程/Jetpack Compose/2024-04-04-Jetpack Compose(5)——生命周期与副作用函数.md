---
layout: post
title: "Jetpack Compose(5)——生命周期与副作用函数"
date:  2024-04-04 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

## 一、 Composable 的生命周期
Composable 组件都是函数，Composable 函数执行会得到一棵视图树，每一个 Composable 组件对应视图树上的一个节点。Composable 的生命周期定义如下：

- **onActive(添加到视图树)** Composable 首次被执行，即在视图树上创建对应的节点。
- **onUpdate(重组)** Composable 跟随重组不断执行，更新视图树上对应的节点。
- **onDispose(从视图树移除)** Composable 不再被执行，对应节点从视图树上移除。

对于 Compose 编写 UI 来说，页面的变化，是依靠状态的变化，Composable 进行重组，渲染出不同的页面。当页面可见时，对应的节点被添加到视图树，当页面不可见时，对应的节点从视图树移除。所以，虽然 Activity 有前后台的概念，但是使用 Compose 编写的页面，对于 Composable 没有前后台切换的概念。当页面切换为不可见时，对应的节点也被立即销毁了，不会像 Activity 或者 Fragment 那样在后台保存实例。

## 二、 Composable 的副作用
上一篇将重组的文章讲到，Composable 重组过程中可能反复执行，并且中间环节有可能被打断，只保证最后一次执行的状态时正确的。
试想一个问题，如果在 Composable 函数中弹一个 Toast ，当 Composable 发生重组时，这个 Toast 会弹多少次，是不是就无法控制了。再比如，在 Composable 函数中读写函数之外的变量，读写文件，请求网络等等，这些操作是不是都无法得到保证了。类似这样，在 Composable 执行过程中，凡是会影响外界的操作，都属于副作用。在 Composable 重组过程中，这些副作用行为都难以得到保证，那怎么办？为了是副作用只发生在生命周期的特定阶段， Compose 提供了一系列副作用函数，来确保行为的可预期性。下面，我们看看这些副作用函数的使用场景。

### 2.1 SideEffect
**SideEffect 在每次成功重组的时候都会执行。**
Composable 在重组过程中会反复执行，但是重组不一定每次都会成功，有的可能会被中断，中途失败。 **SideEffect 仅在重组成功的时候才会执行。**

特点：

1. 重组成功才会执行。
2. 有可能会执行多次。

所以，SideEffect 函数不能用来执行耗时操作，或者只要求执行一次的操作。
典型使用场景，比如在主题中设置状态栏，导航栏颜色等。

```
SideEffect {
    val window = (view.context as Activity).window
    window.statusBarColor = colorScheme.primary.toArgb()
    WindowCompat.getInsetsController(window, view).isAppearanceLightStatusBars = darkTheme
}
```

### 2.2 DisposableEffect
**DisposableEffect 可以感知 Composable 的 `onActive` 和 `onDispose`, 允许使用该函数完成一些预处理和收尾工作。**

典型的使用的场景，注册与取消注册：

```
DisposableEffect(vararg keys: Any?) {
    // register(callback)
    onDispose {
        // unregister(callback)
    }
}
```

这里首先参数 keys 表示，当 keys 变化时， DisposableEffect 会重新执行，如果在整个生命周期内，只想执行一次，则可以传入 `Unit`
而 `onDispose` 代码块则会在 Composable 进入 onDispose 时执行。

### 2.3 LaunchedEffect
LaunchedEffect 用于在 **Composable 中启动协程**，当 Composable 进入 onAtive 时，LaunchedEffect 会自动启动协程，执行 block 中的代码。当 Composable 进入 onDispose 时，协程会自动取消。
使用方法：

```
LaunchedEffect(vararg keys: Any?) {
    // do Something async
}
```

同样支持可观察参数，当 key 变化时，当前协程自动结束，同时开启新协程。

### 2.4 rememberCoroutineScope
LaunchedEffect 只能在 Composable 中调用，如果想**在非 Composable 环境中使用协程**，比如在 Button 的 OnClick 中开启协程，并希望在 Composable 进入 onDispose 时自动取消，则可以使用 `rememberCoroutineScope` 。
具体用法如下：

```
@Composable
fun Test() {
    val scope = rememberCoroutineScope()
    Button(
        onClick = {
            scope.launch {
                // do something
            }
        }
    ) {
        Text("click me")
    }
}
```

DisposableEffect 配合 rememberCoroutineScope 可以实现 LaunchedEffect 同样的效果，但是一般这样做没有什么意义。

### 2.5 rememberUpdatedState
`rememberUpdatedState` 一般和 `DisposableEffect` 或者 `LaunchedEffect` 配套使用。当使用 `DisposableEffect` 或者 `LaunchedEffect` 时，代码块中用到某个值会在外部更新，如何获取到最新的值呢？看一个例子，比如玩王者荣耀时，预选英雄，然后将英雄显示出来，十秒倒计时后，显示最终选择的英雄，倒计时期间，可以改变选择的英雄。

```
@Composable
fun ChooseHero() {
    var sheshou by remember {
        mutableStateOf("狄仁杰")
    }

    Column {
        Text(text = "预选英雄： $sheshou")
        Button(onClick = {
            sheshou = "马可波罗"
        }) {
            Text(text = "改选：马可波罗")
        }
        FinalChoose(sheshou)
    }
}

@Composable
fun FinalChoose(hero: String) {
    var tips by remember {
        mutableStateOf("游戏倒计时：10s")
    }
    LaunchedEffect(key1 = Unit) {
        delay(10000)
        tips = "最终选择的英雄是：$hero"
    }
    Text(text = tips)
}
```

代码运行效果如下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose5/pic1.gif)

我们预选了狄仁杰，倒计时期间，点击 button， 改选马可波罗，最终选择的英雄确显示狄仁杰。
分析原因如下：在 `FinalChoose` 中参数 `hero` 来源于外部，它的值改变，会触发重组，但是，由于 LaunchedEffect 函数，key 赋值 `Unit`, 重组过程中，协程代码块并不会重新执行，感知不到外部的变化。要使能够获取到外部的最新值，一种方式是将 `hero` 作为 LaunchedEffect 的可观察参数。修改代码如下：

```
@Composable
fun FinalChoose(hero: String) {
    var tips by remember {
        mutableStateOf("游戏倒计时：10s")
    }
    LaunchedEffect(key1 = hero) {
        delay(10000)
        tips = "最终选择的英雄是：$hero"
    }
    Text(text = tips)
}
```

此时再次执行，在倒计时期间，我们点击 button， 改变预选英雄，结果显示正常了，最终选择的即为马可波罗。但是该方案并不符合我们的需求，前面讲到， LaunchedEffect 的参数 key，发生变化时，协程会取消，并重新启动新的协程，这意味着，当倒计时过程中，我们改变了 key ， 重新启动的协程能够获取到改变后的值，但是倒计时也重新开始了，这显然不是我们所期望的结果。

而 `rememberUpdatedState` 就是用来解决这种场景的。在不中断协程的情况下，**始终能够获取到最新的值**。看一下 rememberUpdatedState 如何使用。
我们把 LaunchedEffect 的参数 key 还原成 Unit。使用 rememberUpdatedState 定义 currentHero。

```
@Composable
fun FinalChoose(hero: String) {
    var tips by remember {
        mutableStateOf("游戏倒计时：10s")
    }

    val currentHero by rememberUpdatedState(newValue = hero)

    LaunchedEffect(key1 = Unit) {
        delay(10000)
        tips = "最终选择的英雄是：$currentHero"
    }
    Text(text = tips)
}
```

这样，运行结果就符合我们的预期了。

![](/assets/images/技术/编程/Jetpack%20Compose/compose5/pic2.gif)

### 2.6 derivedStateOf
上面的例子中，有一点不完美的地方，游戏倒计时时间没有更新。下面使用 derivedStateOf 来优化这个功能。

```
@Composable
fun FinalChoose(hero: String) {
    var time by remember {
        mutableIntStateOf(10)
    }

    val tips by remember {
        derivedStateOf {
            "游戏倒计时：${time}s"
        }
    }

    LaunchedEffect(key1 = Unit) {
        repeat(10) {
            delay(1000)
            time--
        }
    }
    Text(
        text = if (time == 0) {
            "最终选择的英雄是：$hero"
        } else {
            tips
        }
    )
}
```

![](/assets/images/技术/编程/Jetpack%20Compose/compose5/pic3.gif)

现在效果好多了。这里我们不再需要 rememberUpdatedState 了。首先定义了时间，时一个 Int 类型的 State，然后借助 derivedStateOf 定义 tip ，时一个 String 类型的 State。

**derivedStateOf 的作用是从一个或者多个 State 派生出另一个 State**。如果某个状态是从其他状态对象计算或派生得出的，则可以使用 derivedStateOf。使用此函数可确保仅当计算中使用的状态之一发生变化时才会进行计算。
derivedStateOf 的使用不难，但是和 remember 的配合使用可以有很多玩法来适应不同的场景，主要的关注点还是在触发重组的条件上，这个要综合实际的场景和性能来觉得是用 key 来触发重组还是改变引用的状态来触发重组。

### 2.7 snapshotFlow
前面使用 rememberUpdatedState 可以在 LaunchedEffect 中始终获取到外部状态的最新的值。但是无法感知到状态的变化，也就是说外部状态变化了，LaunchedEffect 中的代码无法第一时间被通知到。用 snapshotFlow 则可以解决这个场景。

**snapshotFlow 用于将一个 `State<T>` 转换成一个协程中的 Flow**。 当 snpashotFlow 块中读取到的 State 对象之一发生变化时，如果新值与之前发出的值不相等，Flow 会向收集器发出最新的值（此行为类似于 Flow.distinctUntilChaned）。
看具体使用：

```
@Composable
fun FinalChoose(hero: String) {
    var time by remember {
        mutableIntStateOf(10)
    }

    var tips by remember {
        mutableStateOf("游戏倒计时：10s")
    }

    LaunchedEffect(key1 = Unit) {
        launch {
            repeat(10) {
                delay(1000)
                time--
            }
        }
        launch {
            snapshotFlow { time }.collect {
                    tips = "游戏倒计时：${it}s"
                }
        }
    }

    Text(
        text = if (time == 0) {
            "最终选择的英雄是：$hero"
        } else {
            tips
        }
    )
}
```

运行结果和上一次一样，这里我们不再使用 `derivedStateOf`, 而是启动了两个协程，一个协程用于倒计时技术，另一个协程则将 time 这个 State 转换成 Flow, 然后进行收集，并更新 tips。

### 2.8 produceState
**`produceState` 用于将任意外部数据源转换为 State**。
比如上面的例子中，我们将倒计时时间定义在 ViewModel 中，并且倒计时的逻辑在 ViewModel 中实现，在 UI 中就可以借助 produceState 来实现。

```
@Composable
fun FinalChoose(hero: String) {
    val time = viewModel.time

    val tips by produceState<String>(initialValue = "游戏倒计时：10s") {
        value = "游戏倒计时：${time}s"

        awaitDispose {
            // 做一些收尾的工作
        }
    }
    Text(
        text = if (time == 0) {
            "最终选择的英雄是：$hero"
        } else {
            tips
        }
    )
}
```

我们看一下 `produceState` 的源码实现：

```
@Composable
fun <T> produceState(
    initialValue: T,
    vararg keys: Any?,
    producer: suspend ProduceStateScope<T>.() -> Unit
): State<T> {
    val result = remember { mutableStateOf(initialValue) }
    @Suppress("CHANGING_ARGUMENTS_EXECUTION_ORDER_FOR_NAMED_VARARGS")
    LaunchedEffect(keys = keys) {
        ProduceStateScopeImpl(result, coroutineContext).producer()
    }
    return result
}
```

很好理解，就是定义了一个状态 State, 然后启动了一个协程，在协程中去更新 State 的值。参数 key 发生变化时，协程会取消，然后重新启动，生成新的 State。
同时注意到，在 produceState 中可以使用 `awaitDispose{ }` 方法做一些收尾工作。这是不是很容易联想到 `callbackFlow` 的使用场景。没错，基于回调的接口实现，利用 `callbackFlow` 很容易转换为协程的 Flow, 而 produceState 即可将其转换为 Compose 中的 State。比如 BroadcastReceiver、ContentProvider、网络请求等等。

```
val currentPerson by produceState<Person?>(null, viewModel) {
    val disposable = viewModel.registerPersonObserver { person ->
        value = person
    }

    awaitDispose {
        disposable.dispose()
    }
}
```

再看一个网络请求的例子：

```
@Composable
fun GetApi(url: String, repository: Repository): Recomposer.State<Result<Data>> {
    return produceState(initialValue = Result.Loading, url, repository) {
        val data = repository.load(url)
        value = if (result == null) {
            Result.Error
        } else {
            Result.Success(data)
        }
    }
}
```

## 三、总结
本文主要介绍了 Composable 的声明周期，以及常用的副作用函数。
在重组过程中，应该极力避免副作用的发生。根据场景，使用合适的副作用函数。

## 写在最后
个人认为 Compose 中最重要的知识域有两个——状态和重组、Modifier 修饰符。经过前面这些文章的讲解，状态和重组基本上主要的知识点都讲到了，知识有一定的前后连贯性。而 Modifier 修饰符庞大的类别体系中，将不再具有这样的关联，可以挨个独立学习。接下来的文章，我将不依次介绍 Modifier 的类别。而是介绍 Android 开发中的应用领域在 Compose 中的处理方式，比如自定义 Layout, 动画，触摸反馈等等，然后在这些知识点中，讲解涉及到的 Modifier。欢迎大家继续关注！