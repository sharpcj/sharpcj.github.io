---
layout: post
title: "Jetpack Compose(4)——重组"
date:  2024-04-03 17:58:12 +0800
categories: ["技术", "编程", "Jetpack Compose"]
tag: ["Kotlin", "Compsoe"]
---

上一篇文章讲了 Compose 中状态管理的基础知识，本文讲解 Compose 中状重组的相关知识。

## 一、状态变化
### 1.1 状态变化是什么
根据上篇文章的讲解，在 Compose 我们使用 State 来声明一个状态，当状态发生变化时，则会触发重组。那么状态变化是指什么呢？
下面我们来看一个例子：

```
@Composable
fun NumList() {
    val num by remember {
        mutableStateOf(mutableListOf(1, 2, 3))
    }
    Column {
        Button(onClick = {
            num += (num.last() + 1)
            Log.d("sharpcj", "num: $num")
        }) {
            Text(text = "click to add one")
        }
        num.forEach {
            Text(text = "item --> $it")
        }
    }
}
```

这段代码中，我们定义了一个 State ，其包裹的类型是 MutableList， 并且每次点击,我们就给该 mutableList 增加一个元素。运行一下：

![](/assets/images/技术/编程/Jetpack%20Compose/compose4/pic1.gif)

我们点击了按钮，界面并没有发生变化，但是，从日志看到，每次点击后，list 中的元素的确增加了一个。

```
2024-03-18 20:51:41.472 12574-12574 sharpcj                 com.sharpcj.hellocompose             D  num: [1, 2, 3, 4]
2024-03-18 20:51:42.411 12574-12574 sharpcj                 com.sharpcj.hellocompose             D  num: [1, 2, 3, 4, 5]
2024-03-18 20:51:43.347 12574-12574 sharpcj                 com.sharpcj.hellocompose             D  num: [1, 2, 3, 4, 5, 6]
```

原因是什么呢？其实状态发生变化，实际上指的是 State 包裹的对象，进行 equals 比较，如果不相等，则认为状态变化，否则认为没有发生变化。所以这里就解释得通了，我们虽然在点击按钮后，给 mutableList 增加了元素，但是 mutableList 在进行前后比较时，比较的是其引用，对象的引用并没有发生变化，所以没有发生重组。【这里结论并不准确，下面稳定类型详细解释说】
那为了让其发生重组，我们稍作修改，每次点击按钮时，创建一个新的 list，然后赋值，看看是不是我们所期待的结果。

```
@Composable
fun NumList() {
    var num by remember {
        mutableStateOf(mutableListOf(1, 2, 3))
    }
    Column {
        Button(onClick = {
            val num1 = num.toMutableList()
            num1 += (num1.last() + 1)
            num = num1
            Log.d("sharpcj", "num: $num")
        }) {
            Text(text = "click to add one")
        }
        num.forEach {
            Text(text = "item --> $it")
        }
    }
}
```

再次运行程序：

![](/assets/images/技术/编程/Jetpack%20Compose/compose4/pic2.gif)

结果符合我们的预期。那对于 List 类型的数据对象，每次状态发生变化，我们创建了一个新对象，这样在进行 equals 比较时，必定不相等，则会触发重组。

### 1.2 mutableStateListOf 和 mutableStateMapOf
上面的问题，我们虽然接解决了, 但是写法不够优雅，其实 Compose 给我们提供了一个函数 mutableStateListOf 来解决这类问题，我们看看这个函数怎么用，改写上面的例子

```
@Composable
fun NumList() {
    val num = remember {
        mutableStateListOf(1, 2, 3)
    }
    Column {
        Button(onClick = {
            num += (num.last() + 1)
            Log.d("sharpcj", "num: $num")
        }) {
            Text(text = "click to add one")
        }
        num.forEach {
            Text(text = "item --> $it")
        }
    }
}
```

这样就可以满足我们的需求。 `mutableStateListOf` 返回了一个可感知内部数据变化的 `SnapshotStateList<T>`, 它的内部的实现为了保证不变性，仍然是拷贝元素，只不过它用了更加高效的实现，比我们单纯用 toMutableList 要高效得多。
由于 SnapshotStateList 继承了 MutableList 接口，使得 MutableList 中定义的方法，依然可以使用。
同理，对于 Map 类型的对象， Compose 中提供了 `mutableStateMapOf` 方法，可以更优雅，更高效地进行处理。

思考如下问题：
假如我定义了一个类型：`data class Hero(var name: String, var age: Int)`, 然后使用 `mutableStateListOf` 定义了状态，其中的元素是自定义的类型 Hero， 当改变 Hero 的属性时， 与该状态相关的 Composable 是否会发生重组？

```
data class Hero(var name: String, var age: Int)

@Composable
fun HeroInfo() {
    val heroList = remember {
        mutableStateListOf(Hero(name = "安其拉", age = 18), Hero(name = "鲁班", age = 19))
    }

    Column {
        Button(onClick = {
            heroList[0].name = "DaJi"
            heroList[0].age = 22
        }) {
            Text(text = "test click")
        }

        heroList.forEach {
            Text(text = "student, name: ${it.name}, age: ${it.age} ")
        }
    }
}
```

## 二、重组的特性
### 2.1 Composable 重组是智能的
传统 View 体系通过修改 View 的私有属性来改变 UI, Compose 则通过重组刷新 UI。 Compose 的重组非常“智能”，当重组发生时，只有状态发生更新的 Composable 才会参与重组，没有变化的 Composable 会跳过本次重组。

```
@Composable
fun KingHonour() {
    Column {
        var name by remember {
            mutableStateOf("周瑜")
        }
        Button(onClick = {
            name = "小乔"
        }) {
            Text(text = "改名")
        }
        Text(text = "鲁班")
        Text(text = name)

    }
}
```

该例子中，点击按钮，改变了 name 的值，触发重组，Button 和 `Text(text = "鲁班")`，并不依赖该状态，虽然在重组时被调用了，但是在运行时并不会真正的执行。因为其参数没有变化，Compose 编译器会在编译器插入相关的比较代码。只有最后一个 Text 依赖该状态，会参与真正的重组。

### 2.2 Composable 会以任意顺序执行

```
@Composable
fun Navi() {
    Box {
        FirstScreen()
        SecondScreen()
        ThirdScreen()
    }
}
```

在代码中出现多个 Composable 函数时，它们并不一定按照在代码中出现的顺序执行，比如在一个 Box 中，处于前景的 UI 具有较高优先级。所以不要试图通过外部变量与其它 Composable 产生关联。

### 2.3 Composable 会并发执行
重组中的 Composable 并不一定执行在 UI 线程，它们可以在后台线程中并发执行，这样利于发挥多喝处理器的优势。正因为此，也需要考虑线程安全问题。

### 2.4 Composable 会反复执行
除了重组会造成 Composable 的再次执行外，在动画等场景中每一帧的变化都可能引起 Composable 的执行。因此 Composable 可能在短时间内多次执行。

### 2.5 Composable 的执行是“乐观”的
所谓“乐观”是指 Composable 最终会依据最新的状态正确地完成重组。在某些场景下，状态可能会连续变化，可能会导致中间态的重组在执行时被打断，新的重组会插进来，对于被打断的重组，Compose 不会将执行一半的结果反应到视图树上，因为最后一次的状态总归是正确的。

## 三、重组范围
原则：重组范围最小化。
只有受到了 State 变化影响的代码块，才会参与到重组，不依赖 State 变化的代码则不参与重组。
如何确定重组范围呢？修改上面的例子：

```
@Composable
fun RecompositionTest() {
    Column {
        Box {
            Log.i("sharpcj", "RecompositionTest - 1")
            Column {
                Log.i("sharpcj", "RecompositionTest - 2")
                var name by remember {
                    mutableStateOf("周瑜")
                }
                Button(onClick = {
                    name = "小乔"
                }) {
                    Log.i("sharpcj", "RecompositionTest - 3")
                    Text(text = "改名")
                }
                Text(text = "鲁班")
                Text(text = name)
            }
        }
        Box {
            Log.i("sharpcj", "RecompositionTest - 4")
        }
        Card {
            Log.i("sharpcj", "RecompositionTest - 5")
        }
    }
}
```

运行，第一次我们看到，打印了如下日志：

```
2024-03-22 15:36:15.303 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 1
2024-03-22 15:36:15.305 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 2
2024-03-22 15:36:15.326 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 3
2024-03-22 15:36:15.337 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 4
2024-03-22 15:36:15.344 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 5
```

这是正常的，每个控件范围内都执行了。我们点击，button， 改变了 name 状态。打印如下日志：

```
2024-03-22 15:37:48.480 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 1
2024-03-22 15:37:48.480 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 2
2024-03-22 15:37:48.491 19870-19870 sharpcj                 com.sharpcj.hellocompose             I  RecompositionTest - 4
```

首先我们 name 这个状态影响的组件时 Text，它所在的作用域应该是 `Column` 内部。打印 R`ecompositionTest - 2` 好理解，可为什么连 Column 的上一级作用域 Box 也被调用了，并且连该 Box 的统计 Box 也被调用了，但是 Card 却又没有被调用。这个好像与上面说的原则相悖。其实不然，我们看看 `Column`、`Box`、`Card` 源码就清楚了。

```
@Composable
inline fun Column(
    modifier: Modifier = Modifier,
    verticalArrangement: Arrangement.Vertical = Arrangement.Top,
    horizontalAlignment: Alignment.Horizontal = Alignment.Start,
    content: @Composable ColumnScope.() -> Unit
) {
    val measurePolicy = columnMeasurePolicy(verticalArrangement, horizontalAlignment)
    Layout(
        content = { ColumnScopeInstance.content() },
        measurePolicy = measurePolicy,
        modifier = modifier
    )
}
@Composable
inline fun Box(
    modifier: Modifier = Modifier,
    contentAlignment: Alignment = Alignment.TopStart,
    propagateMinConstraints: Boolean = false,
    content: @Composable BoxScope.() -> Unit
) {
    val measurePolicy = rememberBoxMeasurePolicy(contentAlignment, propagateMinConstraints)
    Layout(
        content = { BoxScopeInstance.content() },
        measurePolicy = measurePolicy,
        modifier = modifier
    )
}
@Composable
fun Card(
    modifier: Modifier = Modifier,
    shape: Shape = CardDefaults.shape,
    colors: CardColors = CardDefaults.cardColors(),
    elevation: CardElevation = CardDefaults.cardElevation(),
    border: BorderStroke? = null,
    content: @Composable ColumnScope.() -> Unit
) {
    Surface(
        modifier = modifier,
        shape = shape,
        color = colors.containerColor(enabled = true),
        contentColor = colors.contentColor(enabled = true),
        tonalElevation = elevation.tonalElevation(enabled = true),
        shadowElevation = elevation.shadowElevation(enabled = true, interactionSource = null).value,
        border = border,
    ) {
        Column(content = content)
    }
}
```

不难发现， Column 和 Box 都是使用 inline 修饰的。
最后简单了解下 Compose 重组的底层原理。
经过 Compose 编译器处理后的 Composable 代码在对 State 进行读取时，能够自动建立关联，在运行过程中，当 State 变化时， Compose 会找到关联的代码块标记为 Invalid, 在下一渲染帧到来之前，Compose 触发重组并执行 invalid 代码块， invalid 代码块即下一次重组的范围。能够被标记为 Invalid 的代码必须是非 inline 且无返回值的 Composable 函数或 lambda。

需要注意的是，重组的范围，与只能跳过并不冲突，确定了重组范围，会调用对应的组件代码，但是当参数没有变化时，在运行时不会真正执行，会跳过本次重组。

## 四、参数类型的稳定性
### 4.1 稳定和不稳定
前面，Composable 状态变化触发重组，状态变化基于 equals 比较结果，这是不准确的。准确地说：只有当比较的状态对象，是稳定的，才能通过 equals 比较结果确定是否重组。什么叫稳定的？还是看一个例子：

```
data class Hero(var name: String)

val shangDan = Hero("吕布")

@Composable
fun StableTest() {
    var greeting by remember {
        mutableStateOf("hello, 鲁班")
    }

    Column {
        Log.i("sharpcj", "invoke --> 1")
        Text(text = greeting)
        Button(onClick = {
            greeting = "hello, 鲁班大师"
        }) {
            Text(text = "搞错了，是鲁班大师")
        }
        ShangDan(shangDan)
    }
}

@Composable
fun ShangDan(hero: Hero) {
    Log.i("sharpcj", "invoke --> 2")
    Text(text = hero.name)
}
```

运行一下，打印

```
2024-03-22 17:07:50.248 26973-26973 sharpcj                 com.sharpcj.hellocompose             I  invoke --> 1
2024-03-22 17:07:50.272 26973-26973 sharpcj                 com.sharpcj.hellocompose             I  invoke --> 2
```

点击 Button，再次看到打印：

```
2024-03-22 17:07:53.182 26973-26973 sharpcj                 com.sharpcj.hellocompose             I  invoke --> 1
2024-03-22 17:07:53.191 26973-26973 sharpcj                 com.sharpcj.hellocompose             I  invoke --> 2
```

问题来了， Shangdan 这个组件依赖的只依赖一个参数，并且参数也没有改变，为什么确在重组过程中被调用了呢？
接下来，我们将 Hero 这个类做点改变，将其属性声明由 var 变成 val：

```
data class Hero(val name: String)
```

再次运行，

```
2024-03-22 17:35:41.435 28561-28561 sharpcj                 com.sharpcj.hellocompose             I  invoke --> 1
2024-03-22 17:35:41.458 28561-28561 sharpcj                 com.sharpcj.hellocompose             I  invoke --> 2
```

点击button:

```
2024-03-22 17:35:47.790 28561-28561 sharpcj                 com.sharpcj.hellocompose             I  invoke --> 1
```

这次，`Shangdan` 这个 Composable 没有参与重组了。为什么会这样呢？

其实是在因为此前，用 `var` 声明 Hero 类的属性时，Hero 类被 Compose 编译器认为是不稳定类型。即有可能，我们传入的参数引用没有变化，但是属性被修改过了，而 UI 又确实需要显示修改后的最新值。而当用 val 声明属性了，Compose 编译器认为该对象，只要对象引用不要变，那么这个对象就不会发生变化，自然 UI 也就不会发生变化，所以就跳过了这次重组。
常用的基本数据类型以及函数类型(lambda)都可以称得上是稳定类型，它们都不可变。反之，如果状态是可变的，那么比较 equals 结果将不再可信。在遇到不稳定类型时，Compose 的抉择是宁愿牺牲一些性能，也总好过显示错误的 UI。

### 4.2 @Stable 和 @Immutable
上面讲了稳定与不稳定的概念，然而实际开发中，我们经常会根据业务自定义 data class, 难道用了 Compose， 虽然 Kotlin 编码规范，强调尽量使用 `val`, 但是还是要根据实际业务，使用 `var` 来定义可变属性。对于这种类型，我们可以为其添加 @Stable 注解，让编译器将其视为稳定类型。从而发挥智能重组的作用，提升重组的性能。

```
@Stable
data class Hero(var name: String)
```

这样，Hero 即便使用 `var` 声明属性，它作为参数传入 Composable 中，只要对象引用没变，都不会触发重组。所以具体什么时候使用该注解，还需要根据需求灵活使用。

除了 `@Stable`，Compose 还提供了另一个类似的注解 `@Immutable`，与 `@Stable` 不同的是，`@Immutable` 用来修饰的类型应该是完全不可变的。而 `@Stable` 可以用在函数、属性等更多场景。使用起来更加方便，由于功能叠加，未来 `@Immutable` 有可能会被移除，建议优先使用 `@Stable`。

最后总结一下：本文接着上篇文章的状态，讲解了重组的一些特性，如何确定重组的范围，以及重组的中的类型稳定性概念，以及如何提升非稳定类型在重组过程中的性能。
下一篇文章将会讲解 Composable 的生命周期以及重组的副作用函数。