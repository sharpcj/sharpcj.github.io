---
layout: post
title: "Kotlin 协程三——数据流 Flow"
date:  2022-01-17 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin", "flow"]
---

[toc]

## 一、Flow 的基本使用
Kotlin 协程中使用挂起函数可以实现非阻塞地执行任务并将结果返回回来，但是只能返回单个计算结果。但是如果希望有多个计算结果返回回来，则可以使用 Flow。

### 1.1 Sequence 与 Flow
介绍 Flow 之前，先看下序列生成器：

```
val intSequence = sequence<Int> {
        Thread.sleep(1000) // 模拟耗时任务1
        yield(1)
        Thread.sleep(1000) // 模拟耗时任务2
        yield(2)
        Thread.sleep(1000) // 模拟耗时任务3
        yield(3)
    }

intSequence.forEach {
        println(it)
    }
```

如上，取出序列生成器中的值，需要迭代序列生成器，按照我们的预期，依次返回了三个结果。

Sequence 是同步调用，是阻塞的，无法调用其它的挂起函数。显然，我们更多地时候希望能够异步执行多个任务，并将结果依次返回回来，Flow 是该场景的最优解。

Flow 源码如下，只有一个 collect 方法。

```
public interface Flow<out T> {

    @InternalCoroutinesApi
    public suspend fun collect(collector: FlowCollector<T>)
}
```

Flow 可以非阻塞地执行多个任务，并返回多个结果, 在 Flow 中可以调用其它挂起函数。取出 Flow 中的值，则需要调用 collect 方法，Flow 的使用形式为：

```
Flow.collect()  // 伪代码
```

由于 collect 是一个挂起函数， collect 方法必须要在协程中调用。

### 1.2 Flow 的简单使用
实现上述 Sequence 类似的效果：

```
private fun createFlow(): Flow<Int> = flow {
    delay(1000)
    emit(1)
    delay(1000)
    emit(2)
    delay(1000)
    emit(3)
}

fun main() = runBlocking {
    createFlow().collect {
        println(it)
    }
}
```

上述代码使用 `flow{ ... }` 来构建一个 Flow 类型，具有如下特点：

`flow{ ... }` 内部可以调用 suspend 函数；
`createFlow` 不需要 suspend 来标记；(为什么没有标记为挂起函数，去可以调用挂起函数？)
使用 `emit()` 方法来发射数据；
使用 `collect()` 方法来收集结果。

### 1.3 创建常规 Flow 的常用方式
**flow{...}**

```
flow {
    delay(1000)
    emit(1)
    delay(1000)
    emit(2)
    delay(1000)
    emit(3)
}
```

**flowOf()**

```
flowOf(1,2,3).onEach {
    delay(1000)
}
```

`flowOf()` 构建器定义了一个发射固定值集的流, 使用 `flowOf` 构建 Flow 不需要显示调用 `emit()` 发射数据

**asFlow()**

```
listOf(1, 2, 3).asFlow().onEach {
    delay(1000)
}
```

使用 `asFlow()` 扩展函数，可以将各种集合与序列转换为流，也不需要显示调用 `emit()` 发射数据

### 1.4 Flow 是冷流（惰性的）
如同 Sequences 一样， Flows 也是惰性的，即在调用末端流操作符(collect 是其中之一)之前， flow{ ... } 中的代码不会执行。我们称之为冷流。

```
private fun createFlow(): Flow<Int> = flow {
    println("flow started")
    delay(1000)
    emit(1)
    delay(1000)
    emit(2)
    delay(1000)
    emit(3)
}

fun main() = runBlocking {
    val flow = createFlow()
    println("calling collect...")
    flow.collect {
        println(it)
    }
    println("calling collect again...")
    flow.collect {
        println(it)
    }
}
```

结果如下：

```
calling collect...
flow started
1
2
3
calling collect again...
flow started
1
2
3
```

这是一个返回一个 Flow 的函数 `createFlow` 没有标记 `suspend` 的原因，即便它内部调用了 suspend 函数，但是调用 createFlow 会立即返回，且不会进行任何等待。而再每次收集结果的时候，才会启动流。

那么有没有热流呢？ 后面讲的 ChannelFlow 就是热流。只有上游产生了数据，就会立即发射给下游的收集者。

### 1.5 Flow 的取消
流采用了与协程同样的协助取消。流的收集可以在当流在一个可取消的挂起函数（例如 delay）中挂起的时候取消。取消Flow 只需要取消它所在的协程即可。
以下示例展示了当 withTimeoutOrNull 块中代码在运行的时候流是如何在超时的情况下取消并停止执行其代码的：

```
fun simple(): Flow<Int> = flow { 
    for (i in 1..3) {
        delay(100)          
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // 在 250 毫秒后超时
        simple().collect { value -> println(value) } 
    }
    println("Done")
}
```

注意，在 simple 函数中流仅发射两个数字，产生以下输出：

```
Emitting 1
1
Emitting 2
2
Done
```

## 二、 Flow 的操作符
### 2.1 Terminal flow operators 末端流操作符
末端操作符是在流上用于启动流收集的挂起函数。 collect 是最基础的末端操作符，但是还有另外一些更方便使用的末端操作符：

- 转化为各种集合，toList/toSet/toCollection
- 获取第一个（first）值，最后一个（last）值与确保流发射单个（single）值的操作符
- 使用 reduce 与 fold 将流规约到单个值
- count
- launchIn/produceIn/broadcastIn

下面看几个常用的末端流操作符

#### 2.1.1 collect
收集上游发送的数据

#### 2.1.2 reduce
reduce 类似于 Kotlin 集合中的 reduce 函数，能够对集合进行计算操作。前面提到，reduce 是一个末端流操作符。

```
fun main() = runBlocking {
    val sum = (1..5).asFlow().reduce { a, b ->
        a + b
    }
    println(sum)
}
```

输出结果：

```
15
```

#### 2.1.3 fold
fold 也类似于 Kotlin 集合中的 fold，需要设置一个初始值，fold 也是一个末端流操作符。

```
fun main() = runBlocking {
    val sum = (1..5).asFlow().fold(100) { a, b ->
        a + b
    }
    println(sum)
}
```

输出结果：

```
115
```

#### 2.1.4 launchIn
launchIn 用来在指定的 CoroutineScope 内启动 flow， 需要传入一个参数： CoroutineScope
源码：

```
public fun <T> Flow<T>.launchIn(scope: CoroutineScope): Job = scope.launch {
    collect() // tail-call
}
```

示例：

```
private val mDispatcher = Executors.newSingleThreadExecutor().asCoroutineDispatcher()
fun main() {
    val scope = CoroutineScope(mDispatcher)
    (1..5).asFlow().onEach { println(it) }
        .onCompletion { mDispatcher.close() }
        .launchIn(scope)
}
```

输出结果：

```
1
2
3
4
5
```

再看一个例子：

```
fun main() = runBlocking{
    val cosTime = measureTimeMillis {
        (1..5).asFlow()
            .onEach { delay(100) }
            .flowOn(Dispatchers.IO)
            .collect { println(it) }

        flowOf("one", "two", "three", "four", "five")
            .onEach { delay(200) }
            .flowOn(Dispatchers.IO)
            .collect { println(it) }
    }
    println("cosTime: $cosTime")
}
```

我们希望并行执行两个 Flow ，看下输出结果：

```
1
2
3
4
5
one
two
three
four
five
cosTime: 1645
```

结果并不是并行执行的，这个很好理解，因为第一个 collect 不执行完，不会走到第二个。

正确地写法应该是，为每个 Flow 单独起一个协程：

```
fun main() = runBlocking<Unit>{
    launch {
        (1..5).asFlow()
            .onEach { delay(100) }
            .flowOn(Dispatchers.IO)
            .collect { println(it) }
    }
    launch {
        flowOf("one", "two", "three", "four", "five")
            .onEach { delay(200) }
            .flowOn(Dispatchers.IO)
            .collect { println(it) }
    }
}
```

或者使用 launchIn， 写法更优雅：

```
fun main() = runBlocking<Unit>{

    (1..5).asFlow()
        .onEach { delay(100) }
        .flowOn(Dispatchers.IO)
        .onEach { println(it) }
        .launchIn(this)

    flowOf("one", "two", "three", "four", "five")
        .onEach { delay(200) }
        .flowOn(Dispatchers.IO)
        .onEach { println(it) }
        .launchIn(this)
}
```

输出结果：

```
1
one
2
3
4
two
5
three
four
five
```

### 2.2 流是连续的
与 Sequence 类似，Flow 的每次单独收集都是按顺序执行的，除非进行特殊操作的操作符使用多个流。 默认情况下不启动新协程。 从上游到下游每个过渡操作符都会处理每个发射出的值然后再交给末端操作符。

```
fun main() = runBlocking {
    (1..5).asFlow()
        .filter {
            println("Filter $it")
            it % 2 == 0
        }
        .map {
            println("Map $it")
            "string $it"
        }.collect {
            println("Collect $it")
        }
}
```

输出：

```
Filter 1
Filter 2
Map 2
Collect string 2
Filter 3
Filter 4
Map 4
Collect string 4
Filter 5
```

### 2.3 onStart 流启动时
Flow 启动开始执行时的回调，在耗时操作时可以用来做 loading。

```
fun main() = runBlocking {
    (1..5).asFlow()
        .onEach { delay(200) }
        .onStart { println("onStart") }
        .collect { println(it) }
}
```

输出结果：

```
onStart
1
2
3
4
5
```

### 2.4 onCompletion 流完成时
Flow 完成时（正常或出现异常时），如果需要执行一个操作，它可以通过两种方式完成：

#### 2.4.1 使用 try ... finally 实现

```
fun main() = runBlocking {
    try {
        flow {
            for (i in 1..5) {
                delay(100)
                emit(i)
            }
        }.collect { println(it) }
    } finally {
        println("Done")
    }
}
```

2.4.2 通过 onCompletion 函数实现

```
fun main() = runBlocking {
    flow {
        for (i in 1..5) {
            delay(100)
            emit(i)
        }
    }.onCompletion { println("Done") }
        .collect { println(it) }
}
```

输出：

```
1
2
3
4
5
Done
```

### 2.5 Backpressure 背压
Backpressure 是响应式编程的功能之一， Rxjava2 中的 Flowable 支持的 Backpressure 策略有：

- MISSING：创建的 Flowable 没有指定背压策略，不会对通过 OnNext 发射的数据做缓存或丢弃处理。
- ERROR：如果放入 Flowable 的异步缓存池中的数据超限了，则会抛出 MissingBackpressureException 异常。
- BUFFER：Flowable 的异步缓存池同 Observable 的一样，没有固定大小，可以无限制添加数据，不会抛出 MissingBackpressureException 异常，但会导致 OOM。
- DROP：如果 Flowable 的异步缓存池满了，会丢掉将要放入缓存池中的数据。
- LATEST：如果缓存池满了，会丢掉将要放入缓存池中的数据。这一点跟 DROP 策略一样，不同的是，不管缓存池的状态如何，LATEST 策略会将最后一条数据强行放入缓存池中。

而在Flow代码块中，每当有一个处理结果 我们就可以收到，但如果处理结果也是耗时操作。就有可能出现，发送的数据太多了，处理不及时的情况。
Flow 的 Backpressure 是通过 suspend 函数实现的。

#### 2.5.1 buffer 缓冲
buffer 对应 Rxjava 的 BUFFER 策略。 buffer 操作指的是设置缓冲区。当然缓冲区有大小，如果溢出了会有不同的处理策略。

- 设置缓冲区，如果溢出了，则将当前协程挂起，直到有消费了缓冲区中的数据。
- 设置缓冲区，如果溢出了，丢弃最新的数据。
- 设置缓冲区，如果溢出了，丢弃最老的数据。

缓冲区的大小可以设置为 0，也就是不需要缓冲区。

看一个未设置缓冲区的示例，假设每产生一个数据然后发射出去，要耗时 100ms ，每次处理一个数据需要耗时 700ms,代码如下：

```
fun main() = runBlocking {
    val cosTime = measureTimeMillis {
        (1..5).asFlow().onEach {
            delay(100)
            println("produce data: $it")
        }.collect {
            delay(700)
            println("collect: $it")
        }
    }
    println("cosTime: $cosTime")
}
```

结果如下：

```
produce data: 1
collect: 1
produce data: 2
collect: 2
produce data: 3
collect: 3
produce data: 4
collect: 4
produce data: 5
collect: 5
cosTime: 4069
```

由于流是惰性的，且是连续的，所以整个流中的数据处理完成大约需要 4000ms

下面，我们使用 buffer() 设置一个缓冲区。buffer(),
接收两个参数，第一个参数是 size, 表示缓冲区的大小。第二个参数是 BufferOverflow， 表示缓冲区溢出之后的处理策略，其值为下面的枚举类型，默认是 `BufferOverflow.SUSPEND`

处理策略源码如下：

```
public enum class BufferOverflow {
    /**
     * Suspend on buffer overflow.
     */
    SUSPEND,

    /**
     * Drop **the oldest** value in the buffer on overflow, add the new value to the buffer, do not suspend.
     */
    DROP_OLDEST,

    /**
     * Drop **the latest** value that is being added to the buffer right now on buffer overflow
     * (so that buffer contents stay the same), do not suspend.
     */
    DROP_LATEST
}
```

**设置缓冲区，并采用挂起的策略**

修改，代码，我们设置缓冲区大小为 1 ：

```
fun main() = runBlocking {
    val cosTime = measureTimeMillis {
        (1..5).asFlow().onEach {
            delay(100)
            println("produce data: $it")
        }.buffer(2, BufferOverflow.SUSPEND)
            .collect {
                delay(700)
                println("collect: $it")
            }
    }
    println("cosTime: $cosTime")
}
```

结果如下：

```
produce data: 1
produce data: 2
produce data: 3
produce data: 4
collect: 1
produce data: 5
collect: 2
collect: 3
collect: 4
collect: 5
cosTime: 3713
```

可见整体用时大约为 3713ms。buffer 操作符可以使发射和收集的代码并发运行，从而提高效率。
下面简单分析一下执行流程:
这里要注意的是，buffer 的容量是从 0 开始计算的。
首先，我们收集第一个数据，产生第一个数据，然后 2、3、4 存储在了缓冲区。第5个数据发射时，缓冲区满了，会挂起。等到第1个数据收集完成之后，再发射第5个数据。

**设置缓冲区，丢弃最新的数据**

如果上述代码处理缓存溢出策略为 BufferOverflow.DROP_LATEST，代码如下：

```
fun main() = runBlocking {
    val cosTime = measureTimeMillis {
        (1..5).asFlow().onEach {
            delay(100)
            println("produce data: $it")
        }.buffer(2, BufferOverflow.DROP_LATEST)
            .collect {
                delay(700)
                println("collect: $it")
            }
    }
    println("cosTime: $cosTime")
}
```

输出如下：

```
produce data: 1
produce data: 2
produce data: 3
produce data: 4
produce data: 5
collect: 1
collect: 2
collect: 3
cosTime: 2272
```

可以看到，第4个数据和第5个数据因为缓冲区满了直接被丢弃了，不会被收集。

**设置缓冲区，丢弃旧的数据**

如果上述代码处理缓存溢出策略为 BufferOverflow.DROP_OLDEST，代码如下：

```
fun main() = runBlocking {
    val cosTime = measureTimeMillis {
        (1..5).asFlow().onEach {
            delay(100)
            println("produce data: $it")
        }.buffer(2, BufferOverflow.DROP_OLDEST)
            .collect {
                delay(700)
                println("collect: $it")
            }
    }
    println("cosTime: $cosTime")
}
```

输出结果如下：

```
produce data: 1
produce data: 2
produce data: 3
produce data: 4
produce data: 5
collect: 1
collect: 4
collect: 5
cosTime: 2289
```

可以看到，第4个数据进入缓冲区时，会把第2个数据丢弃掉，第5个数据进入缓冲区时，会把第3个数据丢弃掉。

#### 2.5.2 conflate 合并
当流代表部分操作结果或操作状态更新时，可能没有必要处理每个值，而是只处理最新的那个。conflate 操作符可以用于跳过中间值：

```
fun main() = runBlocking {
    val cosTime = measureTimeMillis {
        (1..5).asFlow().onEach {
            delay(100)
            println("produce data: $it")
        }.conflate()
            .collect {
                delay(700)
                println("collect: $it")
            }
    }
    println("cosTime: $cosTime")
}
```

输出结果：

```
produce data: 1
produce data: 2
produce data: 3
produce data: 4
produce data: 5
collect: 1
collect: 5
cosTime: 1596
```

`conflate` 操作符是不设缓冲区，也就是缓冲区大小为 0，丢弃旧数据，也就是采取 DROP_OLDEST 策略，其实等价于 buffer(0, BufferOverflow.DROP_OLDEST) 。

### 2.6 Flow 异常处理
#### 2.6.1 catch 操作符捕获上游异常
前面提到的 onCompletion 用来Flow是否收集完成，即使是遇到异常也会执行。

```
fun main() = runBlocking {
    (1..5).asFlow().onEach {
        if (it == 4) {
            throw Exception("test exception")
        }
        delay(100)
        println("produce data: $it")
    }.onCompletion {
        println("onCompletion")
    }.collect {
        println("collect: $it")
    }
}
```

输出：

```
produce data: 1
collect: 1
produce data: 2
collect: 2
produce data: 3
collect: 3
onCompletion
Exception in thread "main" java.lang.Exception: test exception
...
```

其实在 `onCompletion` 中是可以判断是否有异常的， onCompletion(action: suspend FlowCollector<T>.(cause: Throwable?) -> Unit)` 是有一个参数的，如果 flow 的上游出现异常，这个参数不为 null，如果上游未出现异常，则为 null, 据此，我们可以在 onCompletion 中判断异常：

```
fun main() = runBlocking {
    (1..5).asFlow().onEach {
        if (it == 4) {
            throw Exception("test exception")
        }
        delay(100)
        println("produce data: $it")
    }.onCompletion { cause ->
        if (cause != null) {
            println("flow completed exception")
        } else {
            println("onCompletion")
        }
    }.collect {
        println("collect: $it")
    }
}
```

但是， onCompletion 智能判断是否出现了异常，并不能捕获异常。

捕获异常可以使用 `catch` 操作符。

```
fun main() = runBlocking {
    (1..5).asFlow().onEach {
        if (it == 4) {
            throw Exception("test exception")
        }
        delay(100)
        println("produce data: $it")
    }.onCompletion { cause ->
        if (cause != null) {
            println("flow completed exception")
        } else {
            println("onCompletion")
        }
    }.catch { ex ->
        println("catch exception: ${ex.message}")
    }.collect {
        println("collect: $it")
    }
}
```

输出结果：

```
produce data: 1
collect: 1
produce data: 2
collect: 2
produce data: 3
collect: 3
flow completed exception
catch exception: test exception
```

但是如果把 `onCompletion` 和 `catch` 交换一下位置，则 catch 操作捕获到异常之后，不会再影响下游：
代码：

```
fun main() = runBlocking {
    (1..5).asFlow().onEach {
        if (it == 4) {
            throw Exception("test exception")
        }
        delay(100)
        println("produce data: $it")
    }.catch { ex ->
        println("catch exception: ${ex.message}")
    }.onCompletion { cause ->
        if (cause != null) {
            println("flow completed exception")
        } else {
            println("onCompletion")
        }
    }.collect {
        println("collect: $it")
    }
}
```

输出结果：

```
produce data: 1
collect: 1
produce data: 2
collect: 2
produce data: 3
collect: 3
catch exception: test exception
onCompletion
```

>- catch 操作符用于实现异常透明化处理, catch 只是中间操作符不能捕获下游的异常,。
>- catch 操作符内，可以使用 throw 再次抛出异常、可以使用 emit() 转换为发射值、可以用于打印或者其他业务逻辑的处理等等

#### 2.6.2 retry、retryWhen 操作符重试

```
public fun <T> Flow<T>.retry(
    retries: Long = Long.MAX_VALUE,
    predicate: suspend (cause: Throwable) -> Boolean = { true }
): Flow<T> {
    require(retries > 0) { "Expected positive amount of retries, but had $retries" }
    return retryWhen { cause, attempt -> attempt < retries && predicate(cause) }
}
```

如果上游遇到了异常，并使用了 retry 操作符，则 retry 会让 Flow 最多重试 retries 指定的次数

```
fun main() = runBlocking {
    (1..5).asFlow().onEach {
        if (it == 4) {
            throw Exception("test exception")
        }
        delay(100)
        println("produce data: $it")
    }.retry(2) {
        it.message == "test exception"
    }.catch { ex ->
        println("catch exception: ${ex.message}")
    }.collect {
        println("collect: $it")
    }
}
```

输出结果：

```
produce data: 1
collect: 1
produce data: 2
collect: 2
produce data: 3
collect: 3
produce data: 1
collect: 1
produce data: 2
collect: 2
produce data: 3
collect: 3
produce data: 1
collect: 1
produce data: 2
collect: 2
produce data: 3
collect: 3
catch exception: test exception
```

注意，只有遇到异常了，并且 retry 方法返回 true 的时候才会进行重试。

retry 最终调用的是 retryWhen 操作符。下面的代码与上面的代码逻辑一致。

```
fun main() = runBlocking {
    (1..5).asFlow().onEach {
        if (it == 4) {
            throw Exception("test exception")
        }
        delay(100)
        println("produce data: $it")
    }.retryWhen { cause, attempt ->
        cause.message == "test exception" && attempt < 2
    }.catch { ex ->
        println("catch exception: ${ex.message}")
    }.collect {
        println("collect: $it")
    }
}
```

试想： 如果 将代码中的 catch 和 retry/retryWhen 交换位置，结果如何？

### 2.7 Flow 线程切换
#### 2.7.1 响应线程
Flow 是基于 CoroutineContext 进行线程切换的。因为 Collect 是一个 suspend 函数，必须在 CoroutineScope 中执行，所以响应线程是由 CoroutineContext 决定的。比如，在 Main 线程总执行 collect, 那么响应线程就是 Dispatchers.Main。

#### 2.7.2 flowOn 切换线程
Rxjava 通过 subscribeOn 和 ObserveOn 来决定发射数据和观察者的线程。并且，上游多次调用 subscribeOn 只会以最后一次为准。
而 Flows 通过 flowOn 方法来切换线程，多次调用，都会影响到它上游的代码。举个例子：

```
private val mDispatcher = Executors.newSingleThreadExecutor().asCoroutineDispatcher()

fun main() = runBlocking {
    (1..5).asFlow().onEach {
        printWithThreadInfo("produce data: $it")
    }.flowOn(Dispatchers.IO)
        .map {
            printWithThreadInfo("$it to String")
            "String: $it"
        }.flowOn(mDispatcher)
        .onCompletion {
            mDispatcher.close()
        }
        .collect {
            printWithThreadInfo("collect: $it")
        }
}
```

输出结果如下：

```
thread id: 13, thread name: DefaultDispatcher-worker-1 ---> produce data: 1
thread id: 13, thread name: DefaultDispatcher-worker-1 ---> produce data: 2
thread id: 13, thread name: DefaultDispatcher-worker-1 ---> produce data: 3
thread id: 13, thread name: DefaultDispatcher-worker-1 ---> produce data: 4
thread id: 13, thread name: DefaultDispatcher-worker-1 ---> produce data: 5
thread id: 12, thread name: pool-1-thread-1 ---> 1 to String
thread id: 12, thread name: pool-1-thread-1 ---> 2 to String
thread id: 1, thread name: main ---> collect: String: 1
thread id: 12, thread name: pool-1-thread-1 ---> 3 to String
thread id: 1, thread name: main ---> collect: String: 2
thread id: 12, thread name: pool-1-thread-1 ---> 4 to String
thread id: 1, thread name: main ---> collect: String: 3
thread id: 12, thread name: pool-1-thread-1 ---> 5 to String
thread id: 1, thread name: main ---> collect: String: 4
thread id: 1, thread name: main ---> collect: String: 5
```

可以看到，发射数据是在 Dispatchers.IO 线程执行的， map 操作时在我们自定义的线程池中进行的，collect 操作在 Dispatchers.Main 线程进行。

### 2.8 Flow 中间转换操作符
#### 2.8.1 map
前面例子已经有用到 map 操作符，map 操作符不止可以用于 Flow, map 操作符勇于 List 表示将 List 中的每个元素转换成新的元素，并添加到一个新的 List 中，最后再讲新的 List 返回，
map 操作符用于 Flow 表示将流中的每个元素进行转换后再发射出来。

```
fun main() = runBlocking {
    (1..5).asFlow().map { "string: $it" }
        .collect {
            println(it)
        }
}
```

输出：

```
string: 1
string: 2
string: 3
string: 4
string: 5
```

#### 2.8.2 transform
在使用 transform 操作符时，可以任意多次调用 emit ，这是 transform 跟 map 最大的区别:

```
fun main() = runBlocking {
    (1..5).asFlow().transform {
        emit(it * 2)
        delay(100)
        emit("String: $it")
    }.collect {
            println(it)
        }
}
```

输出结果：

```
2
String: 1
4
String: 2
6
String: 3
8
String: 4
10
String: 5
```

#### 2.8.3 onEach
遍历

```
fun main() = runBlocking {
    (1..5).asFlow()
        .onEach { println("onEach: $it") }
        .collect { println(it) }
}
```

输出：

```
onEach: 1
1
onEach: 2
2
onEach: 3
3
onEach: 4
4
onEach: 5
5
```

#### 2.8.4 filter
按条件过滤

```
fun main() = runBlocking {
    (1..5).asFlow()
        .filter { it % 2 == 0 }
        .collect { println(it) }
}
```

输出结果：

```
2
4
```

#### 2.8.5 drop / dropWhile
- drop 过滤掉前几个元素
- dropWhile 过滤满足条件的元素

#### 2.8.6 take
take 操作符只取前几个 emit 发射的值

```
fun main() = runBlocking {
    (1..5).asFlow().take(2).collect {
            println(it)
        }
}
```

输出：

```
1
2
```

#### 2.8.7 zip
zip 是可以将2个 flow 进行合并的操作符

```
fun main() = runBlocking {
    val flowA = (1..6).asFlow()
    val flowB = flowOf("one", "two", "three","four","five").onEach { delay(200)
    flowA.zip(flowB) { a, b -> "$a and $b" }
        .collect {
            println(it)
        }
}
```

输出结果：

```
1 and one
2 and two
3 and three
4 and four
5 and five
```

zip 操作符会把 flowA 中的一个 item 和 flowB 中对应的一个 item 进行合并。即使 flowB 中的每一个 item 都使用了 delay() 函数，在合并过程中也会等待 delay() 执行完后再进行合并。

如果 flowA 和 flowB 中 item 个数不一致，则合并后新的 flow item 个数，等于较小的 item 个数

```
fun main() = runBlocking {
    val flowA = (1..5).asFlow()
    val flowB = flowOf("one", "two", "three","four","five", "six", "seven").onEach { delay(200) }
    flowA.zip(flowB) { a, b -> "$a and $b" }
        .collect {
            println(it)
        }
}
```

输出结果：

```
1 and one
2 and two
3 and three
4 and four
5 and five
```

#### 2.8.8 combine
combine 也是合并，但是跟 zip 不太一样。

使用 combine 合并时，每次从 flowA 发出新的 item ，会将其与 flowB 的最新的 item 合并。

```
fun main() = runBlocking {
    val flowA = (1..5).asFlow().onEach { delay(100) }
    val flowB = flowOf("one", "two", "three","four","five", "six", "seven").onEach { delay(200) }
    flowA.combine(flowB) { a, b -> "$a and $b" }
        .collect {
            println(it)
        }
}
```

输出结果：

```
1 and one
2 and one
3 and one
3 and two
4 and two
5 and two
5 and three
5 and four
5 and five
5 and six
5 and seven
```

#### 2.8.9 flattenConcat 和 flattenMerge 扁平化处理
**flattenConcat**
flattenConcat 将给定流按顺序展平为单个流，而不交错嵌套流。
源码：

```
@FlowPreview
public fun <T> Flow<Flow<T>>.flattenConcat(): Flow<T> = flow {
    collect { value -> emitAll(value) }
}
```

例子：

```
fun main() = runBlocking {
    val flowA = (1..5).asFlow()
    val flowB = flowOf("one", "two", "three","four","five").onEach { delay(1000) }

    flowOf(flowA,flowB)
        .flattenConcat()
        .collect{ println(it) }
}
```

输出：

```
1
2
3
4
5
// delay 1000ms
one
// delay 1000ms
two
// delay 1000ms
three
// delay 1000ms
four
// delay 1000ms
five
```

**flattenMerge**
fattenMerge 有一个参数，并发限制，默认位 16。
源码：

```
@FlowPreview
public fun <T> Flow<Flow<T>>.flattenMerge(concurrency: Int = DEFAULT_CONCURRENCY): Flow<T> {
    require(concurrency > 0) { "Expected positive concurrency level, but had $concurrency" }
    return if (concurrency == 1) flattenConcat() else ChannelFlowMerge(this, concurrency)
}
```

可见，参数必须大于0， 并且参数为 1 时，与 flattenConcat 一致。

```
fun main() = runBlocking {
    val flowA = (1..5).asFlow().onEach { delay(100) }
    val flowB = flowOf("one", "two", "three","four","five").onEach { delay(200) }

    flowOf(flowA,flowB)
        .flattenMerge(2)
        .collect{ println(it) }
}
```

输出结果：

```
1
one
2
3
two
4
5
three
four
five
```

#### 2.8.10 flatMapMerge 和 flatMapConcat
flatMapMerge 由 map、flattenMerge 操作符实现

```
@FlowPreview
public fun <T, R> Flow<T>.flatMapMerge(
    concurrency: Int = DEFAULT_CONCURRENCY,
    transform: suspend (value: T) -> Flow<R>
): Flow<R> = map(transform).flattenMerge(concurrency)
```

例子：

```
fun main() = runBlocking {
    (1..5).asFlow()
        .flatMapMerge {
            flow {
                emit(it)
                delay(1000)
                emit("string: $it")
            }
        }.collect { println(it) }
}
```

输出结果：

```
1
2
3
4
5
// delay 1000ms
string: 1
string: 2
string: 3
string: 4
string: 5
```

flatMapConcat由 map、flattenConcat 操作符实现

```
@FlowPreview
public fun <T, R> Flow<T>.flatMapConcat(transform: suspend (value: T) -> Flow<R>): Flow<R> = map(transform).flattenConcat()
```

例子：

```
fun main() = runBlocking {
    (1..5).asFlow()
        .flatMapConcat {
            flow {
                emit(it)
                delay(1000)
                emit("string: $it")
            }
        }.collect { println(it) }
}
```

输出结果：

```
1
// delay 1000ms
string: 1
2
// delay 1000ms
string: 2
3
// delay 1000ms
string: 3
4
// delay 1000ms
string: 4
5
// delay 1000ms
string: 5
```

>flatMapMerge 和 flatMapConcat 都是将一个 flow 转换成另一个流。
>区别在于： flatMapMerge 不会等待内部的 flow 完成 ， 而调用 flatMapConcat 后，collect 函数在收集新值之前会等待 flatMapConcat 内部的 flow 完成。

#### 2.8.11 flatMapLatest
当发射了新值之后，上个 flow 就会被取消。

```
fun main() = runBlocking {
    (1..5).asFlow().onEach { delay(100) }
        .flatMapLatest {
            flow {
                println("begin flatMapLatest $it")
                delay(200)
                emit("string: $it")
                println("end flatMapLatest $it")
            }
        }.collect {
            println(it)
        }
}
```

输出结果：

```
begin flatMapLatest 1
begin flatMapLatest 2
begin flatMapLatest 3
begin flatMapLatest 4
begin flatMapLatest 5
end flatMapLatest 5
string: 5
```

## 三、 StateFlow 和 SharedFlow
StateFlow 和 SharedFlow 是用来替代 BroadcastChannel 的新的 API。用于上游发射数据，能同时被多个订阅者收集数据。

### 3.1 StateFlow
官方文档解释：StateFlow 是一个状态容器式可观察数据流，可以向其收集器发出当前状态更新和新状态更新。还可通过其 value 属性读取当前状态值。如需更新状态并将其发送到数据流，请为 MutableStateFlow 类的 value 属性分配一个新值。

在 Android 中，StateFlow 非常适合需要让可变状态保持可观察的类。

StateFlow有两种类型: StateFlow 和 MutableStateFlow :

```
public interface StateFlow<out T> : SharedFlow<T> {
   public val value: T
}

public interface MutableStateFlow<out T>: StateFlow<T>, MutableSharedFlow<T> {
   public override var value: T
   public fun compareAndSet(expect: T, update: T): Boolean
}
```

状态由其值表示。任何对值的更新都会反馈新值到所有流的接收器中。

#### 3.1.1 StateFlow 基本使用
使用示例：

```
class Test {
    private val _state = MutableStateFlow<String>("unKnown")
    val state: StateFlow<String> get() = _state

    fun getApi(scope: CoroutineScope) {
        scope.launch {
            val res = getApi()
            _state.value = res
        }
    }

    private suspend fun getApi() = withContext(Dispatchers.IO) {
        delay(2000) // 模拟耗时请求
        "hello, stateFlow"
    }
}

fun main() = runBlocking<Unit> {
    val test: Test = Test()

    test.getApi(this) // 开始获取结果

    launch(Dispatchers.IO) {
        test.state.collect {
            printWithThreadInfo(it)
        }
    }
    launch(Dispatchers.IO) {
        test.state.collect {
            printWithThreadInfo(it)
        }
    }
}
```

结果输出如下，并且程序是没有停下来的。

```
thread id: 14, thread name: DefaultDispatcher-worker-3 ---> unKnown
thread id: 12, thread name: DefaultDispatcher-worker-1 ---> unKnown
// 等待两秒
thread id: 14, thread name: DefaultDispatcher-worker-3 ---> hello, stateFlow
thread id: 12, thread name: DefaultDispatcher-worker-1 ---> hello, stateFlow
```

StateFlow 的使用方式与 LiveData 类似。
MutableStateFlow 是可变类型的，即可以改变 value 的值。 StateFlow 则是只读的。这与 LiveData、MutableLiveData一样。为了程序的封装性。一般对外暴露不可变的只读变量。

输出结果证明：

1. StateFlow 是发射的数据可以被在不同的协程中，被多个接受者同时收集的。
2. StateFlow 是热流，只要数据发生变化，就会发射数据。

程序没有停下来，因为在 StateFlow 的收集者调用 collect 会挂起当前协程，而且永远不会结束。

StateFlow 与 LiveData 的不同之处：

1. StateFlow 必须有初始值，LiveData 不需要。
2. LiveData 会与 Activity 声明周期绑定，当 View 进入 STOPED 状态时， LiveData.observer() 会自动取消注册，而从 StateFlow 或任意其他数据流收集数据的操作并不会停止。

#### 3.1.2 为什么使用 StateFlow
我们知道 LiveData 有如下特点：

1. 只能在主线程更新数据，即使在子线程通过 postValue()方法，最终也是将值 post 到主线程调用的 setValue()
2. LiveData 是不防抖的
3. LiveData 的 transformation 是在主线程工作
4. LiveData 需要正确处理 “粘性事件” 问题。

鉴于此，使用 StateFlow 可以轻松解决上述场景。

#### 3.1.3 防止任务泄漏
解决办法有两种：

1. 不直接使用 StateFlow 的收集者，使用 asLiveData() 方法将其转换为 LiveData 使用。————为何不直接使用 LiveData, 有毛病？
2. 手动取消 StateFlow 的订阅者的协程，在 Android 中，可以从 Lifecycle.repeatOnLifecycle 块收集数据流。
对应代码如下：

```
lifecycleSope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        test.state.collect {
            printWithThreadInfo(it)
        }
    }
}
```

##### 3.1.4 SateFlow 只会发射最新的数据给订阅者。
我们修改上面代码：

```
class Test {
    private val _state = MutableStateFlow<String>("unKnown")
    val state: StateFlow<String> get() = _state

    fun getApi1(scope: CoroutineScope) {
        scope.launch {
            delay(2000)
            _state.value = "hello，coroutine"
        }
    }

    fun getApi2(scope: CoroutineScope) {
        scope.launch {
            delay(2000)
            _state.value = "hello, kotlin"
        }
    }
}

fun main() = runBlocking<Unit> {
    val test: Test = Test()

    test.getApi1(this) // 开始获取结果
    delay(1000)
    test.getApi2(this) // 开始获取结果

    val job1 = launch(Dispatchers.IO) {
        delay(8000)
        test.state.collect {
            printWithThreadInfo(it)
        }
    }
    val job2 = launch(Dispatchers.IO) {
        delay(8000)
        test.state.collect {
            printWithThreadInfo(it)
        }
    }

    // 避免任务泄漏，手动取消
    delay(10000)
    job1.cancel()
    job2.cancel()
}
```

现在的场景是，先请求 getApi1(), 一秒之后再次请求 getApi2(), 这样 stateFlow 的值加上初始值，一共被赋值过 3 次。确保，三次赋值都完成后，我们再收集 StateFlow 中的数据。
输出结果如下：

```
thread id: 13, thread name: DefaultDispatcher-worker-2 ---> hello, kotlin
thread id: 12, thread name: DefaultDispatcher-worker-1 ---> hello, kotlin
```

结果显示了，StateFlow 只会将最新的数据发射给订阅者。对比 LiveData, LiveData 内部有 version 的概念，对于注册的订阅者，会根据 version 进行判断，将历史数据发送给订阅者。即所谓的“粘性”。我不认为 “粘性” 是 LiveData 的设计缺陷，我认为这是一种特性，有很多场景确实需要用到这种特性。StateFlow 则没有此特性。

那总不能需要用到这种特性的时候，我又使用 LiveData 吧？下面要说的 SharedFlow 就是用来解决此种场景的。

### 3.2 SharedFlow
如果只是需要管理一系列状态更新(即事件流)，而非管理当前状态.则可以使用 SharedFlow 共享流。如果对发出的一连串值感兴趣，则这API十分方便。相比 LiveData 的版本控制，SharedFlow 则更灵活、更强大。

SharedFlow 也有两种类型：SharedFlow 和 MutableSharedFlow：

```
public interface SharedFlow<out T> : Flow<T> {
   public val replayCache: List<T>
}

interface MutableSharedFlow<T> : SharedFlow<T>, FlowCollector<T> {
   suspend fun emit(value: T)
   fun tryEmit(value: T): Boolean
   val subscriptionCount: StateFlow<Int>
   fun resetReplayCache()
}
```

SharedFlow 是一个流，其中包含可用作原子快照的 replayCache。每个新的订阅者会先从replay cache中获取值，然后才收到新发出的值。

MutableSharedFlow可用于从挂起或非挂起的上下文中发射值。顾名思义，可以重置 MutableSharedFlow 的 replayCache。而且还将订阅者的数量作为 Flow 暴露出来。

实现自定义的 MutableSharedFlow 可能很麻烦。因此，官方提供了一些使用 SharedFlow 的便捷方法：

```
public fun <T> MutableSharedFlow(
   replay: Int,   // 当新的订阅者Collect时，发送几个已经发送过的数据给它
   extraBufferCapacity: Int = 0,  // 减去replay，MutableSharedFlow还缓存多少数据
   onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND  // 缓存溢出时的处理策略，三种 丢掉最新值、丢掉最旧值和挂起
): MutableSharedFlow<T>
```

MutableSharedFlow 的参数解释在上面对应的注释中。

#### 3.2.1 SharedFlow 基本使用

```
class SharedFlowTest {
    private val _state = MutableSharedFlow<Int>(replay = 3, extraBufferCapacity = 2)
    val state: SharedFlow<Int> get() = _state

    fun getApi(scope: CoroutineScope) {
        scope.launch {
            for (i in 0..20) {
                delay(200)
                _state.emit(i)
                println("send data: $i")
            }
        }
    }
}

fun main() = runBlocking<Unit> {
    val test: SharedFlowTest = SharedFlowTest()

    test.getApi(this) // 开始获取结果

    val job = launch(Dispatchers.IO) {
        delay(3000)
        test.state.collect {
            println("---collect1: $it")
        }
    }
    delay(5000)
    job.cancel()  // 取消任务， 避免泄漏
}
```

输出结果如下：

```
send data: 0
send data: 1
send data: 2
send data: 3
send data: 4
send data: 5
send data: 6
send data: 7
send data: 8
send data: 9
send data: 10
send data: 11
send data: 12
send data: 13
---collect1: 11
---collect1: 12
---collect1: 13
send data: 14
---collect1: 14
send data: 15
---collect1: 15
send data: 16
---collect1: 16
send data: 17
---collect1: 17
send data: 18
---collect1: 18
send data: 19
---collect1: 19
send data: 20
---collect1: 20
```

分析一下该结果：
SharedFlow 每 200ms 发射一次数据，总共发射 21 个数据出来，耗时大约 4s。
SharedFlow 的 replay 设置为 3, extraBufferCapacity 设置为2， 即 SharedFlow 的缓存为 5 。缓存溢出的处理策略是默认挂起的。
订阅者是在 3s 之后开始手机数据的。此时应该已经发射了 14 个数据，即 0-13， SharedFlow 的缓存为 8， 缓存的数据为 9-13， 但是，只给订阅者发送 3 个旧数据，即订阅者收集到的值是从 11 开始的。

#### 3.2.2 MutableSharedFlow 的其它接口
MutableSharedFlow 还具有 `subscriptionCount` 属性，其中包含处于活跃状态的收集器的数量，以便相应地优化业务逻辑。
MutableSharedFlow 还包含一个 `resetReplayCache` 函数，在不想重放已向数据流发送的最新信息的情况下使用。

### 3.3 StateFlow 和 SharedFlow 的使用场景
StateFlow 的命名已经说明了适用场景， StateFlow 只会向订阅者发射最新的值，适用于对状态的监听。
SharedFlow 可以配置对历史发射的数据进行订阅，适合用来处理对于事件的监听。

### 3.4 将冷流转换为热流
使用 `sharedIn` 方法可以将 Flow 转换为 SharedFlow。详细介绍见下文。