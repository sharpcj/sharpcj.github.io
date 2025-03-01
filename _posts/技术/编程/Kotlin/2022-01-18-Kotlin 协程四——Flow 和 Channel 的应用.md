---
layout: post
title: "Kotlin 协程四——Flow 和 Channel 的应用"
date:  2022-01-18 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin", "flow", "channel"]
---

[toc]

## 一、 Flow 与 Channel 的相互转换
### 1.1 Flow 转换为 Channel
#### 1.1.1 ChannelFlow

```
@InternalCoroutinesApi
public abstract class ChannelFlow<T>(
    // upstream context
    @JvmField public val context: CoroutineContext,
    // buffer capacity between upstream and downstream context
    @JvmField public val capacity: Int,
    // buffer overflow strategy
    @JvmField public val onBufferOverflow: BufferOverflow
) : FusibleFlow<T> {
    ...


    public open fun produceImpl(scope: CoroutineScope): ReceiveChannel<T> =
        scope.produce(context, produceCapacity, onBufferOverflow, start = CoroutineStart.ATOMIC, block = collectToFun)

    ...

}
```

前面提到 ChannelFlow 是热流。只要上游产生数据，就会立即发射给下游收集者。
ChannelFlow 是一个抽象类，并且被标记为内部 Api，不应该在外部代码直接使用。
注意到它内部有一个方法 `produceImpl` 返回的是一个 ReceiveChannel，它的实现是收集上游发射的数据，然后发送到 Channel 中。
有此作为基础。我们可以 调用 `asChannelFlow` 将 Flow 转换 ChannelFlow， 进而转换成 Channel 。

#### 1.1.2 produceIn —— 将 Flow 转换为单播式 Channel
produceIn() 转换创建了一个produce 协程来 collect 原Flow，因此该produce协程应该在恰当时候被关闭或者取消。转换后的 Channel 拥有处理背压的能力。其基本使用方式如下：

```
fun main() = runBlocking {
    val flow = flow<Int> {
        repeat(5) {
            delay(500)
            emit(it)
        }
    }

    val produceIn = flow.produceIn(this)
    for (ele in produceIn) {
        println(ele)
    }
}
```

输出结果：

```
0
1
2
3
4
```

查看 produceIn 源码：

```
@FlowPreview
public fun <T> Flow<T>.produceIn(scope: CoroutineScope): ReceiveChannel<T> = asChannelFlow().produceImpl(scope)
```

#### 1.1.3 broadcastIn —— 将 Flow 转换为广播式 BroadcastChannel。
broadcastIn 转换方式与 produceIn 转换方式实现原理一样，区别是创建出来的 BroadcastChannel。
源码如下：

```
public fun <T> Flow<T>.broadcastIn(
    scope: CoroutineScope,
    start: CoroutineStart = CoroutineStart.LAZY
): BroadcastChannel<T> {
    // Backwards compatibility with operator fusing
    val channelFlow = asChannelFlow()
    val capacity = when (channelFlow.onBufferOverflow) {
        BufferOverflow.SUSPEND -> channelFlow.produceCapacity
        BufferOverflow.DROP_OLDEST -> Channel.CONFLATED
        BufferOverflow.DROP_LATEST ->
            throw IllegalArgumentException("Broadcast channel does not support BufferOverflow.DROP_LATEST")
    }
    return scope.broadcast(channelFlow.context, capacity = capacity, start = start) {
        collect { value ->
            send(value)
        }
    }
}
```

使用方式见上文 BroadcastChannel。

和 BroadcastChannel 一样，broadcastIn 也标记为过时的 API, 不建议继续使用了。

### 1.2 Channel 转换为 Flow
#### 1.2.1 consumeAsFlow/receiveAsFlow —— 将单播式 Channel 转换为 Flow
使用 `consumeAsFlow()/receiveAsFlow()` 将 Channel 转换为 Flow

```
fun main() = runBlocking<Unit> {
    val testChannel = Channel<String>()

    val testFlow = testChannel.receiveAsFlow()

    launch {
        testFlow.collect {
            println(it)
        }
    }

    delay(100)
    testChannel.send("hello")
    delay(100)
    testChannel.send("coroutine")
    delay(100)

    testChannel.close() // 注意只有 Channel 关闭了，协程才能结束
}
```

查看源码：

```
public fun <T> ReceiveChannel<T>.consumeAsFlow(): Flow<T> = ChannelAsFlow(this, consume = true)

public fun <T> ReceiveChannel<T>.receiveAsFlow(): Flow<T> = ChannelAsFlow(this, consume = false)

private class ChannelAsFlow<T>(
    private val channel: ReceiveChannel<T>,
    private val consume: Boolean,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = Channel.OPTIONAL_CHANNEL,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlow<T>(context, capacity, onBufferOverflow) {}
```

`consumeAsFlow` 和 `receiveAsFlow` 都是调用 `ChannelAsFlow` 将 Channel 转换成了 ChannelFlow，所以转换结果是热流。但它们传递的第二个参数 `consume` 不一样。两者区别如下：

- 使用 `consumeAsFlow()` 转换成的 Flow 只能有一个收集器收集，如果有多个收集器收集，将会抛出如下异常：

```
Exception in thread "main" java.lang.IllegalStateException: ReceiveChannel.consumeAsFlow can be collected just once
```

- 使用 `receiveAsFlow()` 转换成的 Flow 可以有多个收集器收集，但是保证每个元素只能被一个收集器收集到，即单播式。
通俗点说，就是使用 `consumeAsFlow()` 只能有一个消费者。 使用 `receiveAsFlow()` 可以有多个消费者，但当向 Channel 中发射一个数据之后，收到该元素的消费者是不确定的。

#### 1.2.2 asFlow —— 将广播式 BroadcastChannel 转换为 Flow
与单播式相对的就是广播式，让每个消费者都收到该元素，这就需要一个广播式的 Chanel：BroadcastChanel。
BroadcastChannel 调用 `asFlow()` 方法即可将其转换为 Flow。

由于该方法也被标记为过时了，替代方案有 SharedFlow 和 StateFlow。

## 二、SharedIn —— 将冷数据流转换为热数据流
将 flow 转换为 SharedFlow,可以使用 SharedIn 方法：

```
public fun <T> Flow<T>.shareIn(
    scope: CoroutineScope,
    started: SharingStarted,
    replay: Int = 0
): SharedFlow<T> {
    ...
}
```

参数解释：

- CoroutineScope 用于共享数据流的 CoroutineScope。此作用域函数的生命周期应长于任何使用方，以使共享数据流在足够长的时间内保持活跃状态
- replay 每个新收集器的数据项数量
- started “启动” 方式

启动方式有：

```
public fun interface SharingStarted {
    public companion object {
        // 立即启动，并且永远不会自动停止
        - public val Eagerly: SharingStarted = StartedEagerly() 

        // 第一个订阅者注册后启动，并且永远不会自动停止
        - public val Lazily: SharingStarted = StartedLazily() 

        // 第一个订阅者注册后启动，最后一个订阅者取消注册后停止
        - public fun WhileSubscribed(
                    stopTimeoutMillis: Long = 0,
                    replayExpirationMillis: Long = Long.MAX_VALUE
                ): SharingStarted =
                    StartedWhileSubscribed(stopTimeoutMillis, replayExpirationMillis)
    }
}
```

## 三、callbackFlow —— 将基于回调的 API 转换为数据流
Kotlin 协程和 Flow 可以完美解决异步调用、线程切换的问题。设计接口时，可以类似 Rxjava 那样，避免使用回调。比如 Room 在内的很多库已经支持将协程用于数据流操作。对于那些还不支持的库，也可以将任何基于回调的 API 转换为协程。

`callbackFlow` 是一个数据流构建器，可以将基于回调的 API 转换为数据流。

### 3.1 callbackFlow 的使用
举例：

```
interface Result<T>  {
    fun onSuccess(t: T)
    fun onFail(msg: String)
}

fun getApi(res: Result<String>) {
    thread{
        printWithThreadInfo("getApiSync")
        Thread.sleep(1000) // 模拟耗时任务
        res.onSuccess("hello")
    }.start()
}
```

getApi() 是一个基于回调设计的接口。如何使用 callbackFlow 转换为 Flow 呢？

```
fun getApi(): Flow<String> = callbackFlow {
    val res = object: Result<String> {
        override fun onSuccess(t: String) {
            trySend(t)
            close(Exception("completion"))
        }

        override fun onFail(msg: String) {
        }
    }
    getApi(res)

    // 一定要调用骨气函数 awaitClose， 保证流一直运行。在`awaitClose` 中移除 API 订阅，防止任务泄漏。
    awaitClose {
        println("close")
    }
}

// 新的 Api 使用方式
fun main() = runBlocking<Unit> {
    getApi().flowOn(Dispatchers.IO)
        .catch {
            println("getApi fail, cause: ${it.message}")
        }.onCompletion {
            println("onCompletion")
        }.collect {
            printWithThreadInfo("getApi success, result: $it")
        }
}
```

这时候你可能有疑问了，这在流的内部不还是使用了基于接口的调用吗，分明没有更方便。看下面的例子，就能体会到了。

### 3.2 callbackFlow 实战
Android 开发中有一个常见的场景：输入关键字进行查询。比如有个 EditText，输入文字后，基于输入的文字进行网络请求或者数据库查询。
假设查询数据的接口：

```
fun <T>query(keyWord: String): Flow<T> {
    return flow {
        //...
    }
}
```

首先定义一个方法将 EditText 内容变化的回调转换成 Flow

```
fun textChangeFlow(editText: EditText): Flow<String> = callbackFlow {
    val watcher = object : TextWatcher {
        override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {

        }

        override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {
        }

        override fun afterTextChanged(s: Editable?) {
            s?.let {
                trySend(s.toString()) 
            }
            
        }
    }
    editText.addTextChangedListener(watcher)
    awaitClose {
        editText.removeTextChangedListener(watcher)
    }
}
```

使用：

```
scope.launch{
    textChangeFlow(editText)
            .debounce(300) // 防抖处理
            .flatMapLatest { keyWord ->  // 只对最新的值进行搜索
                flow {
                    <String>query(keyWord)
                }
            }.collect {
                // ... 处理最终结果
            }
}
```

在这个过程中，我们可以充分使用 Flow 的各种变换，对我们的中间过程进行处理。实现一些很难实现的需求。