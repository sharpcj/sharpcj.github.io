---
layout: post
title: "Jetpack Compose(3) —— 状态管理"
date:  2024-03-13 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

上一篇文章拿 TextField 组件举例时，提到了 State，即状态。本篇文章，即讲解 State 的相关改概念。

## 一、什么是状态
与其它声明式 UI 框架一样，Compose 的职责非常单纯，仅作为对数据状态的反应。如果数据状态没有改变，则 UI 永远不会自行改变。在 Compose 中，每一个组件都是一个被 @Composable 修饰的函数，其状态就是函数的参数，当参数不变，则函数的输出就不会变，唯一的参数决定唯一输出。反言之，如果要让界面发生变化，则需要改变界面的状态，然后 Composable 响应这种变化。
下面还是拿个例子来说，做一个简单的计数器，有一个显示计数的控件，一个增加的按钮，每点击一次，则技术计数器加 1 ，一个减少的按钮，每点击一次，计时器减 1。
假如我们用此前的 View 视图体系，来写这个方法。代码大概像下面这样：

```
class MainActivity : AppCompatActivity() {
    // ...
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...
        binding.incrementBtn.setOnClickListener {
            binding.tvCounter.text = "${Integer.valueOf(binding.tvCounter.text.toString()) + 1 }"
        }

        binding.decrementBtn.setOnClickListener {
            binding.tvCounter.text = "${Integer.valueOf(binding.tvCounter.text.toString()) - 1 }"
        }
    }
}
```

显然上面这个代码，计数逻辑和 UI 的耦合度就很高。稍微优化一下：

```
class MainActivity : AppCompatActivity() {
    // ...
    private var counter: Int = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...
        binding.incrementBtn.setOnClickListener {
            counter++
            updateCounter()
        }

        binding.decrementBtn.setOnClickListener {
            counter--
            updateCounter()
        }
    }

    private fun updateCounter() {
        binding.tvCounter.text = "$counter"
    }
}
```

这个代码的改动主要在于，新增了 counter 用于计数，本质上属于一种 “状态上提”， 原本 TextView 内部的状态 “mText”， 上提到了 Activity 中，这样，即使更换了计数器的 UI， 计数逻辑依然可以复用。

但是当前的代码，仍然有一些问题，比如计数逻辑在 Activity 中，无法到其它页面进行复用，进一步使用 MVVM 结构进行改造。引入 ViewModel, 将状态从 Activity 中上提到 ViewModel 中。

```
class CounterViewModel: ViewModel() {
    private var _counter: MutableStateFlow<Int> = MutableStateFlow(0)
    val counter: StateFlow<Int> get() = _counter

    fun incrementCounter() {
        _counter.value++
    }

    fun decrementCounter() {
        _counter.value--
    }
}

class MainActivity : AppCompatActivity() {
    // ...
    private val viewModel: CounterViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...
        binding.incrementBtn.setOnClickListener {
            viewModel.incrementCounter()
        }

        binding.decrementBtn.setOnClickListener {
            viewModel.decrementCounter()
        }

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.counter.collect {
                    binding.tvCounter.text = $it
                }
            }
        }
    }
}
```

有 Jetpack 库使用经验的应该非常熟悉上面的代码，将状态上提到 ViewModel 中，使用 StateFlow 或者 LiveData 包装起来，在 Ativity 中监听状态的变化，从而自动刷新 UI。

下面，我们在 Compose 中实现上述计数器：

```
@Composable
fun CounterPage() {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        var counter = 0
        Text(text = "$counter")
        Button(onClick = { counter++ }) {
            Text(text = "increment")
        }
        Button(onClick = { counter-- }) {
            Text(text = "decrement")
        }
    }
}
```

我们写出上面的代码，运行。

![](/assets/images/技术/编程/Jetpack%20Compose/compose3/pic1.gif)

结果发现，无论怎么点击，Text 显示的值总是 0 ，我们的计数逻辑没有生效。为了说明这个问题，现在增加一点日志：

```
@Composable
fun CounterPage() {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        var counter = 0
        Log.d("sharpcj", "counter text --> $counter")
        Text(text = "$counter")
        Button(onClick = {
            Log.d("sharpcj", "increment button click ")
            counter++
        }) {
            Text(text = "increment")
        }
        Button(onClick = {
            Log.d("sharpcj", "decrement button click ")
            counter--
        }) {
            Text(text = "decrement")
        }
    }
}
```

再次运行，点击按钮，看到日志如下：

```
2024-03-12 21:39:27.530 21949-21949 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 21:39:30.859 21949-21949 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 21:39:31.309 21949-21949 sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 21:39:31.468 21949-21949 sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 21:39:31.762 21949-21949 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 21:39:31.927 21949-21949 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 21:39:32.661 21949-21949 sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
```

我们重新捋一捋，Compose 的组件实际上就是一个个函数，Compose 刷新 UI 的逻辑是，状态发生变化,触发了重组，函数被重新调用，然后由于参数发生了变化，函数输出改变了，最终渲染出的的画面才会发生变化。
再看上面的代码，我们期望是定义 `counter` 作为了 Text 组件的状态，点击 Button，改变 `counter`, 到这里都没有问题，那么问题处在了哪里呢？问题主要是 `counter` 发生了变化，没有触发重组，即函数没有被重新调用，日志也证明了这一点。
回看我们上面传统 View 视图的写法，此前，我们改变了状态，需要主动调用 `updateCounter` 方法去刷新 UI, 后面经过改造，我们把状态提升到 ViewModel 中，不论是使用 StateFlow 还是使用 LiveData 包装后，我们都需要在 Activity 中监听状态的变化，才能对状态的变化做出响应。针对上面的例子，我们现在清楚了，计数器不生效原因在于 counter 改变后，Compose 没有感知到，没有触发重组。下面需要开始学习 Compose 中的状态了。

## 二、Compsoe 中的状态 State
### 2.1 State
如同传统试图中，需要使用 StateFlow 或者 LiveData 将状态变量包装成一个可观察类型的对象。Compose 中也提供了可观察的状态类型，可变状态类型 MutableState 和 不可变状态类型 State。我们需要使用 State/MutableState 将状态变量包装起来，这样即可触发重组。更为方便的是，声明式 UI 框架中，不需要我们显示注册监听状态变化，框架自动实现了这一订阅关系。我们来改写上面的代码：

```
@Composable
fun CounterPage() {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        val counter: MutableState<Int> = mutableStateOf(0)
        Log.d("sharpcj", "counter text --> ${counter.value}")
        Text(text = "${counter.value}")
        Button(onClick = {
            Log.d("sharpcj", "increment button click ")
            counter.value++
        }) {
            Text(text = "increment")
        }
        Button(onClick = {
            Log.d("sharpcj", "decrement button click ")
            counter.value--
        }) {
            Text(text = "decrement")
        }
    }
}
```

我们使用了 `mutableStateOf()` 方法初始化了一个 MutableState 类型的状态变量，并传入默认值 0 ，使用的时候，需要调用 counter.value。
再次运行，结果发现，点击按钮，计数器值还是没有变化，日志如下：

```
2024-03-12 21:57:24.773  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 21:57:31.428  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 21:57:31.437  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 21:57:31.825  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 21:57:31.834  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 21:57:33.047  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 21:57:33.055  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 21:57:33.216  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 21:57:33.224  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 21:57:33.634  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 21:57:33.643  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 21:57:33.792  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 21:57:33.801  6791-6791  sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
```

和上一次不一样了，这次发现，点击按钮之后， `Text(text = "${counter.value}")` 有重新执行，即发生了重组，但是执行的时候，参数没有改变，依然是 0，其实这里涉及到一个重组作用域的概念，就是重组是有一个范围的，关于重组作用范围，稍后再讲。这里需要知道，发生了重组，`Text(text = "${counter.value}")` 有重新执行，那么 `val counter: MutableState<Int> = mutableStateOf(0)` 也有重新执行，相当于重组时，counter 被重新初始化了，并赋予了默认值 0 。所以点击按钮发生了重组，但是计数器的值没有发生改变。要解决这个问题，则需要使用到 Compose 中的一个重要函数 `remember`。

### 2.2 remember
我们先看看 remember 函数的源码：

```
/**
 * Remember the value produced by [calculation]. [calculation] will only be evaluated during the composition.
 * Recomposition will always return the value produced by composition.
 */
@Composable
inline fun <T> remember(crossinline calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(false, calculation)
```

remember 方法的作用是，对其包裹起来的变量值进行缓存，后续发生重组过程中，不会重新初始化，而是直接从缓存中取。具体使用如下：

```
@Composable
fun CounterPage() {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        val counter: MutableState<Int> = remember { mutableStateOf(0) }
        Log.d("sharpcj", "counter text --> ${counter.value}")
        Text(text = "${counter.value}")
        Button(onClick = {
            Log.d("sharpcj", "increment button click ")
            counter.value++
        }) {
            Text(text = "increment")
        }
        Button(onClick = {
            Log.d("sharpcj", "decrement button click ")
            counter.value--
        }) {
            Text(text = "decrement")
        }
    }
}
```

再次运行，这次终于正常了。

![](/assets/images/技术/编程/Jetpack%20Compose/compose3/pic2.gif)

看日志也正确了。每次点击都出发了重组，并且 counter 的值也没有重新初始化。

```
2024-03-12 22:18:53.744 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 0
2024-03-12 22:19:10.397 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 22:19:10.421 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 1
2024-03-12 22:19:10.967 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 22:19:10.981 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 2
2024-03-12 22:19:11.181 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 22:19:11.195 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 3
2024-03-12 22:19:11.649 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 22:19:11.663 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 4
2024-03-12 22:19:11.806 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 22:19:11.821 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 5
2024-03-12 22:19:12.364 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 22:19:12.377 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 4
2024-03-12 22:19:12.640 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 22:19:12.657 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 3
2024-03-12 22:19:13.204 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  increment button click 
2024-03-12 22:19:13.220 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 4
2024-03-12 22:19:13.747 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  decrement button click 
2024-03-12 22:19:13.761 19790-19790 sharpcj                 com.sharpcj.hellocompose             D  counter text --> 3
```

上面的代码中，我们创建 State 的方法如下：

```
val counter: MutableState<Int> = remember { mutableStateOf(0) }
```

使用时，通过 `counter.value` 来使用，这样的代码看起来就很繁琐，我们可以进一步精简写法。
首先， Kotlin 支持类型推导，所以可以写成下面这样：

```
val counter = remember { mutableStateOf(0) }
```

另外，借助于 Kotlin 委托语法，Compose 实现了委托方式赋值，使用 by 关键字即可，用法如下：

```
var counter by remember { mutableStateOf(0) }
```

并导入如下方法：

```
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
```

在使用时，直接使用 `counter++` 和 `counter--，

需要注意的一点是，没有使用委托方式创建的对象，类型是 `MutableState` 类型，我们用 `val` 声明，使用委托方式创建对象，对象类型是 MutableState 包装的对象类型，这里由于赋初始值为 0 ，根据类型推导，counter 就是 `Int` 型，由于要修改 counter 的值，所以须使用 `var` 将其声明为一个可变类型对象。

### 2.3 rememberSaveable
使用 remember 虽然解决了重组过程中，状态被重新初始化的问题，但是当 Activity 销毁重建时，状态值依然会重新初始化，比如横竖屏旋转，UiMode 切换等场景。在传统试图体系中，也存在这样的问题，对此的解决方案有很多，比如重写 Activity 的回调方法，在合适的时机，对数据进行保存和恢复，又或者使用 ViewModel 存放数据，这些方法对于 Compose 当然也有效，但是考虑到在使用 Compose 时，应该弱化 Activity 生命周期的概念，所以前者不适合在 Compose 中使用，而使用 ViewModel 依然是一种优秀的选择，后文再介绍。但是把所有的数据都放到 ViewModel 中，是否是最好的呢，这个要根据具体场景，进行甄别。举个例子，
针对这种场景，Compose 提供了 `rememberSaveable` 这个方法来解决这种场景的问题。

```
var counter by rememberSaveable { mutableStateOf(0) }
```

用法与 remember 方法用法类似，区别在于，rememberSaveable 在横竖屏旋转，UiMode 切换等场景中，能够对其包裹的数据进行缓存。那是否说明 rememberSaveable 可以在所有的场景替换 remember ， remember 方法就没用了？ rememberSaveable 方法比 remember 方法功能更强劲，代价就是性能要差一些，具体使用根据实际场景来选择。

到这里，状态相关的知识点，应该就很清楚了，再回头看上一篇文章中的 TextField 组件，应该能明白为什么那样写了。

## 三、 Stateless 和 Stateful
声明式 UI 的组件一般都可以分为 Stateless 组件和 Stateful 组件。
所谓 stateless 是指这个组件除了依赖参数以外，不依赖其它任何状态。比如 Text 组件,

```
Text("Hello, Compose")
```

相对的，某个组件除了参数以外，还持有或者访问了外部的状态，称为 stateful 组件。比如上一篇文章中提到的 TextField 组件，

```
var text by remember { mutableStateOf("文本框初始值") }
TextField(value = text, onValueChange = {
    text = it
})
```

Stateless 是不依赖于外部状态，仅依赖传入进来的参数，它是一个“纯函数”，即唯一输入，对应唯一输出。也就是参数不变，UI 就不会变化，它的重组只能是来自上层的调用，因此 Compose 编译器对其进行了优化，当 Stateless 的参数没有变化时，它就不会参与重组，重组的范围局限于 Stateless 外部。另外 Stateless 不耦合任何业务，功能更纯粹，所以复用性更好，也更容易测试。
基于此，我们应该尽可能地将 stateful 组件改造成 stateless 组件，这个过程称之为状态上提。

### 3.1 状态上提
状态上提，通常的做法就是将内部状态移除，以参数的形式传入。以及需要回调给调用方的事件，也以参数形式传入。
还是以上面计数器的代码为例，为了简洁，去掉前面添加的 log, 代码如下：

```
@Composable
fun CounterPage() {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        var counter by remember{ mutableStateOf(0) }
        Text(text = "$counter")
        Button(onClick = {
            counter++
        }) {
            Text(text = "increment")
        }
        Button(onClick = {
            counter--
        }) {
            Text(text = "decrement")
        }
    }
}
```

这里计数器主要是依赖了内部状态 counter， 同时两个按钮的点击事件，会改变 counter。状态上提之后，该方法如下：

```
@Composable
fun CounterPage(counter: Int, onIncrement: () -> Unit, onDecrement: () -> Unit) {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text(text = "$counter")
        Button(onClick = {
            onIncrement()
        }) {
            Text(text = "increment")
        }
        Button(onClick = {
            onDecrement()
        }) {
            Text(text = "decrement")
        }
    }
}
```

这样，Counter 组件，就变成了 stateless 组件，不再与业务耦合，职责更加单一，可复用性和可测试性都更强了。此外，状态上提，有助于单一数据源模型的打造。

## 四、状态管理
我们再来看一下在 Compose 中应该如何管理状态。

### 4.1 使用 stateful 管理状态
简单的的 UI 状态，且与业务无关的状态，适合在 Compose 中直接管理。
比如我有一个菜单列表，点一开关，展开一个菜单，再点一下，收起菜单，列表的状态，仅由点击开关这一单一事件决定。并且，列表的状态与任何外部业务无关。那么这种就适合在 Compose 内部进行管理。

### 4.2 使用 StateHolder 管理状态
当业务有一定的复杂度之后，我们可以将业务逻辑相关的状态统一封装到一个 StateHoler 进行管理。剥离 Ui 逻辑，让 Composable 专注 UI 布局。

### 4.3 使用 ViewModel 管理状态
从某种意义上讲，ViewModel 也是一种特殊的 StateHolde。单因为它是保存在 ViewModelStore 中，所以有一下特点：

- 存活范围大，可以脱离 Composition 存在，被所有 Composable 共享。
- 存活时间长，不会因为横竖屏切换或者 UiMode 切换导致数据丢失。

因此，ViewModel 适合管理应用程序全局状态，而且 ViewModel更倾向于管理哪些非 UI 的业务状态。

以上管理方式可以同时使用，结合具体的业务灵活搭配。

### 4.4 LiveData、Rxjava、Flow 转 State
在 MVVM 架构中，使用 ViewModel 来管理状态，如果是新项目，把状态直接定义 State 类型就可以了。

对于传统试图项目，一般使用 LiveData、Rxjava 或者 Flow 这类响应式数据框架。而在 Compose 中需要 State 触发重组，刷新 UI，也有相应的方法，将上述响应式数据流转换为 Compose 中的 State。当上有数据变化时，可以驱动 Composable 完成重组。具体方法如下：

|拓展方法|依赖库|
|---|---|
|LiveData.observeAsState()|androidx.compose:runtime-livedata|
|Flow.collectAsState()|不依赖三方库，Compose 自带|
|Observable.subscribeAsState()|androidx.compose:runtime-rxjava2 或者 androidx.compose:runtime-rxjava3|

## 五、小结
本文主要讲解了 Compose 中状态的概念。最后做个小结，

- Compose UI 依赖状态变化，触发重组，驱动界面更新。
- 使用 remember 和 rememberSaveable 进行状态持久化。remember 保证在 recompose 过程中状态稳定，rememberSaveable 保证 Activity 自动销毁重建过程中状态稳定。
- 状态上提，尽可能将 Stateful 组件转换为 Stateless 组件。
- 视情况使用 Stateful、StateHoler、ViewModel 管理状态。
- 将 LiveData、RxJava、Flow 数据流转换为 State。