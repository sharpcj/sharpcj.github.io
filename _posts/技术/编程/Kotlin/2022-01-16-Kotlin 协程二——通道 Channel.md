---
layout: post
title: "Kotlin 协程二——通道 Channel"
date:  2022-01-16 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin", "channel"]
---

[toc]

## 一、 Channel 基本使用
### 1.1 Channel 的概念
Channel 翻译过来为通道或者管道，实际上就是个队列, 是一个面向多协程之间数据传输的 BlockQueue，用于协程间通信。Channel 允许我们在不同的协程间传递数据。形象点说就是不同的协程可以往同一个管道里面写入数据或者读取数据。它是一个和 BlockingQueue 非常相似的概念。区别在于：BlockingQueue 使用 `put` 和 `take` 往队列里面写入和读取数据，这两个方法是阻塞的。而 Channel 使用 `send` 和 `receive` 两个方法往管道里面写入和读取数据。这两个方法是非阻塞的挂起函数，鉴于此，Channel 的 `send` 和 `receive` 方法也只能在协程中使用。

### 1.2 Channel 的简单使用

```
val channel = Channel<Int>()
launch {
    // 这里可能是消耗大量 CPU 运算的异步逻辑，我们将仅仅做 5 次整数的平方并发送
    for (x in 1..5) channel.send(x * x)
}
// 这里我们打印了 5 次被接收的整数：
repeat(5) { println(channel.receive()) }
println("Done!")
```

输出结果：

```
1
4
9
16
25
Done!
```

### 1.3 Channel 的迭代
如果要取出 Channel 中所有的数据，可以使用迭代。

```
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) {
            channel.send(x * x)
        }
    }

    val iterator = channel.iterator()
    while (iterator.hasNext()) {
        val next = iterator.next()
        println(next)
    }
    println("Done!")
}
```

可以简化成：

```
val channel = Channel<Int>()
launch {
    // 这里可能是消耗大量 CPU 运算的异步逻辑，我们将仅仅做 5 次整数的平方并发送
    for (x in 1..5) channel.send(x * x)
}
for (y in channel) {
    println(y)
}
println("Done!")
```

此时输出结果：

```
1
4
9
16
25
```

最后一行 `Done!` 没有打印出来，并且程序没有结束。此时，我们发现，这种方式，实际上是我们一直在等待读取 Channel 中的数据，只要有数据到了，就会被读取到。

### 1.4 close 关闭 Channel
我们可以使用 `close()` 方法关闭 Channel，来表明没有更多的元素将会进入通道。

```
val channel = Channel<Int>()
launch {
    // 这里可能是消耗大量 CPU 运算的异步逻辑，我们将仅仅做 5 次整数的平方并发送
    for (x in 1..5) channel.send(x * x)
    channle.close()  // 结束发送
}
for (y in channel) {
    println(y)
}
println("Done!")
```

从概念上来讲，调用 close 方法就像向通道发送了一个特殊的关闭指令，这个迭代停止，说明关闭指令已经被接收了。所以这里能够保证所有先前发送出去的原色都能在通道关闭前被接收到。
对于一个 Channel，如果我们调用了它的 close，它会立即停止接受新元素，也就是说这时候它的 isClosedForSend 会立即返回 true，而由于 Channel 缓冲区的存在，这时候可能还有一些元素没有被处理完，所以要等所有的元素都被读取之后 isClosedForReceive 才会返回 true。
输出结果:

```
1
4
9
16
25
Done!
```

### 1.5 Channel 是热流
Flow 是冷流，只有调用末端流操作的时候，上游才会发射数据，与 Flow 不同，Channel 是热流，不管有没有订阅者，上游都会发射数据。

## 二、Channel 的类型
### 2.1 SendChannel 和 ReceiveChannel
Channel 是一个接口，它继承了 `SendChannel` 和 `ReceiveChannel` 两个接口

```
public interface Channel<E> : SendChannel<E>, ReceiveChannel<E>
```

**SendChannel**
`SendChannel` 提供了发射数据的功能，有如下重点接口：

- send 是一个挂起函数，将指定的元素发送到此通道，在该通道的缓冲区已满或不存在时挂起调用者。如果通道已经关闭，调用发送时会抛出异常。
- trySend 如果不违反其容量限制，则立即将指定元素添加到此通道，并返回成功结果。否则，返回失败或关闭的结果。
- close 关闭通道。
- isClosedForSend 判断通道是否已经关闭，如果关闭，调用 send 会引发异常。

**ReceiveChannel**
`ReceiveChannel` 提供了接收数据的功能，有如下重点接口：

- receive 如果此通道不为空，则从中检索并删除元素；如果通道为空，则挂起调用者；如果通道为接收而关闭，则引发ClosedReceiveChannel异常。
- tryReceive 如果此通道不为空，则从中检索并删除元素，返回成功结果；如果通道为空，则返回失败结果；如果通道关闭，则返回关闭结果。
- receiveCatching 如果此通道不为空，则从中检索并删除元素，返回成功结果；如果通道为空，则返回失败结果；如果通道关闭，则返回关闭的原因。
- isEmpty 判断通道是否为空
- isClosedForReceive 判断通道是否已经关闭，如果关闭，调用 receive 会引发异常。
- cancel(cause: CancellationException? = null) 以可选原因取消接收此频道的剩余元素。此函数用于关闭通道并从中删除所有缓冲发送的元素。
- iterator() 返回通道的迭代器

### 2.2 创建不同类型的 Channel
Kotlin 协程库中定义了多个 Channel 类型，所有channel类型的receive方法都是同样的行为: 如果channel不为空, 接收一个元素, 否则挂起。
它们的主要区别在于：

- 内部可以存储元素的数量
- send 是否可以被挂起

Channel 的不同类型：

- Rendezvous channel: 0尺寸buffer (默认类型).
- Unlimited channel: 无限元素, send不被挂起.
- Buffered channel: 指定大小, 满了之后send挂起.
- Conflated channel: 新元素会覆盖旧元素, receiver只会得到最新元素, send永不挂起.

创建 Channel：

```
val rendezvousChannel = Channel<String>()
val bufferedChannel = Channel<String>(10)
val conflatedChannel = Channel<String>(CONFLATED)
val unlimitedChannel = Channel<String>(UNLIMITED)
```

## 三、协程间通过 Channel 实现通信
### 3.1 多个协程访问同一个 Channel
在协程外部定义 Channel, 就可以多个协程可以访问同一个channel，达到协程间通信的目的。

```
fun main() = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x)
    }
    launch {
        delay(10)
        for (y in channel) {
            println(" 1 --> $y")
        }
    }
    launch {
        delay(20)
        for (y in channel) {
            println(" 2 --> $y")
        }
    }
    launch {
        delay(30)
        for (x in 90..100) channel.send(x)
        channel.close()
    }
}
```

### 3.2 produce 和 actor
在协程外部定义 Channel，多个协程同时访问 Channel, 就可以实现生产者消费者模式。

**produce**

使用 produce 可以更便捷地构造生产者

```
fun main() = runBlocking<Unit> {
    val receiveChannel: ReceiveChannel<Int> = GlobalScope.produce {
        var i = 0
        while(true){
            delay(1000)
            send(i)
            i++
        }
        delay(3000)
        receiveChannel.cancel()
    }
}
```

我们可以通过 produce 这个方法启动一个生产者协程，并返回一个 ReceiveChannel，其他协程就可以拿着这个 Channel 来接收数据了。

**actor**

actor 可以用来构建一个消费者协程

```
fun main() = runBlocking<Unit> {
    val sendChannel: SendChannel<Int> = actor<Int> {
        for (ele in channel)
            ele
        }
    }
    
    delay(2000)
    sendChannel.close()
}
```

>注意：不要在循环中使用 receive ，思考为什么？

produce 和 actor 与 launch 一样都被称作“协程启动器”。通过这两个协程的启动器启动的协程也自然的与返回的 Channel 绑定到了一起，因此 Channel 的关闭也会在协程结束时自动完成，以 produce 为例，它构造出了一个 ProducerCoroutine 的对象

### 3.3 扇入和扇出
多个协程可能会从同一个channel中接收值，这种情况称为Fan-out。
多个协程可能会向同一个channel发射值，这种情况称为Fan-in。

### 3.4 BroadcastChannel
#### 3.4.1 BroadcastChannel 基本使用
3.1 中例子提到一对多的情形，从数据处理本身来讲，有多个接收端的时候，同一个元素只会被一个接收端读到。而 BroadcastChannel 则不然，多个接收端不存在互斥现象。

```
public interface BroadcastChannel<E> : SendChannel<E> {

    public fun openSubscription(): ReceiveChannel<E>

    public fun cancel(cause: CancellationException? = null)

    @Deprecated(level = DeprecationLevel.HIDDEN, message = "Binary compatibility only")
    public fun cancel(cause: Throwable? = null): Boolean
}

public fun <E> BroadcastChannel(capacity: Int): BroadcastChannel<E> =
when (capacity) {
    0 -> throw IllegalArgumentException("Unsupported 0 capacity for BroadcastChannel")
    UNLIMITED -> throw IllegalArgumentException("Unsupported UNLIMITED capacity for BroadcastChannel")
    CONFLATED -> ConflatedBroadcastChannel()
    BUFFERED -> ArrayBroadcastChannel(CHANNEL_DEFAULT_CAPACITY)
    else -> ArrayBroadcastChannel(capacity)
}
```

**创建 BroadcastChannel**
创建 BroadcastChannel 需要指定缓冲区大小

```
val broadcastChannel = broadcastChannel<Int>(5)
```

**订阅 broadcastChannel**
订阅 broadcastChannel，那么只需要调用

```
val receiveChannel = broadcastChannel.openSubscription()
```

这样我们就得到了一个 `ReceiveChannel`，获取订阅的消息，只需要调用它的 receive。

#### 3.4.2 使用拓展函数转换
使用 Channel 的拓展函数，也可以将一个 Channel 转换成 BroadcastChannel, 需要指定缓冲区大小。

```
val channel = Channel<Int>()
val broadcast = channel.broadcast(3)
```

这样发射给原 channel 的数据会被读取后发射给转换后的 broadcastChannel。如果还有其他协程也在读这个原始的 Channel，那么会与 BroadcastChannel 产生互斥关系。

#### 3.4.3 过时的 API
BroadcastChannel 源码中的说明：

>Note: This API is obsolete since 1.5.0. It will be deprecated with warning in 1.6.0 and with error in 1.7.0. It is replaced with SharedFlow.

BroadcastChannel 对于广播式的任务来说有点太复杂了。使用通道进行状态管理时会出现一些逻辑上的不一致。例如，可以关闭或取消通道。但由于无法取消状态，因此在状态管理中无法正常使用！
所以官方决定启用 BroadcastChannel。BroadcastChannel 被标记为过时了，在 kotlin 1.6.0 版本中使用将显示警告，在 1.7.0 版本中将显示错误。请使用 SharedFlow 和 StateFlow 替代它。
关于 SharedFlow 和 StateFlow 将在下文中讲到。