---
layout: post
title: "Kotlin 之 let、with、run、apply、also 函数的使用"
date:  2022-04-16 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin"]
---

## 一、内联拓展函数 let
let 扩展函数的实际上是一个作用域函数，当你需要去定义一个变量在一个特定的作用域范围内，let函数的是一个不错的选择；let函数另一个作用就是可以避免写一些判断null的操作。

### 1.1 let 函数的使用的一般结构

```
object.let {
    it.todo() //在函数体内使用it替代object对象去访问其公有的属性和方法
    ...
}

//另一种用途 判断object为null的操作
object?.let { //表示object不为null的条件下，才会去执行let函数体
    it.todo()
}
```

### 1.2 let函数底层的inline扩展函数+lambda结构

```
@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

意思就是 T 类型的对象调用 let 方法，实际调用的是传入 let 方法的 lambda 表达式的 block 块，最终返回 lambda 表达式的返回值。
lambda 表达式内部通过 it 指代该对象。

### 1.3 let 函数常见的适用的场景
- 场景一: 最常用的场景就是使用let函数处理需要针对一个可null的对象统一做判空处理。
- 场景二: 然后就是需要去明确一个变量所处特定的作用域范围内可以使用

```
obj?.funA()
obj?.funB()
obj?.funC()

obj?.let {
    it.funA()
    it.funB()
    it.funC()
}
```

## 二、内联函数 with
### 2.1 with 函数使用的一般结构

```
with(object) {
    //todo
}
```

### 2.2 with 函数底层的inline扩展函数+lambda 结构

```
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

注意，这个 with 函数不是拓展函数，它接收两个参数，第一个参数是要是用的对象，第二个参数是一个 lambda 表达式，该方法实际调用的是第一个参数对象，进行 block 块的调用,
最终返回 lambda 表达式的返回值。
lambda 表达式内部通过 this 指代该对象。

### 2.3 with 函数的适用的场景
适用于调用同一个类的多个方法时，可以省去类名重复，直接调用类的方法即可，经常用于Android中RecyclerView中onBinderViewHolder中，数据model的属性映射到UI上。

```
obj.funA()
obj.funB()
obj.funC()

with(obj) {
    this.funA()
    funB() // this 可省略
    funC)
}
```

## 三、 内联拓展函数 run
### 3.1 run 函数使用的一般结构

```
object.run {
    // todo
}
```

### 3.2 run 函数的inline+lambda 结构

```
@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

run 函数实际上可以说是let和with两个函数的结合体，run函数只接收一个lambda函数为参数，以闭包形式返回，即返回 lambda 表达式的返回值。

### 3.3 run 函数的适用场景

```
obj?.funA()
obj?.funB()
obj?.funC()

obj?.run {
    this.funA()
    funB() // this 可省略
    funC)
}
```

## 四、内联拓展函数 apply
### 4.1 apply 函数使用的一般结构

```
object.apply {
    // todo
}
```

### 4.2 apply 函数的inline+lambda结构

```
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

从结构上来看apply函数和run函数很像，唯一不同点就是它们各自返回的值不一样，run函数是以闭包形式返回最后一行代码的值，而apply函数的返回的是传入对象的本身。

## 五、内联扩展函数 also
### 5.1 also 函数使用的一般结构

```
object.also {
    // todo
}
```

### 5.2 also 函数的inline+lambda结构

```
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```

also函数的结构实际上和let很像唯一的区别就是返回值的不一样，let是以闭包的形式返回，返回函数体内最后一行的值，如果最后一行为空就返回一个Unit类型的默认值。而also函数返回的则是传入对象的本身。

## 六、比较总结
|函数名|定义 inline 的结构|函数体内使用的对象|返回值|是否是扩展函数|
|---|---|---|---|---|
|let|fun T.let(block: (T) -> R): R = block(this)|it指代当前对象|闭包形式返回|是|
|with|fun with(receiver: T, block: T.() -> R): R = receiver.block()|this指代当前对象或者省略|闭包形式返回|否|
|run|fun T.run(block: T.() -> R): R = block()|this指代当前对象或者省略|闭包形式返回|是|
|apply|fun T.apply(block: T.() -> Unit): T|this指代当前对象或者省略|返回this|是|
|also|fun T.also(block: (T) -> Unit): T|it指代当前对象|返回this|是|

## 七、实用例子————Kotlin 实现单例模式
Kotlin 实现单例模式相对 java 来说很简单。比如通过 `object`， `by lazy` 操作，相信大家都会。但有时候，我们想要在单例初始化的时候顺便做一下其它初始化，极有可能会还需要传入参数。
使用 java 时，我最喜欢的实现单例模式是静态内部类的方式，但在 Android 中经常在初始化的时候需要传入 context ，然后选择了双重检查锁方式。
先看 java 代码：

```
public class Singleton {
    private Singleton() {
    }

    /**
     * volatile is since JDK5
     */
    private static volatile Singleton sSingleton;

    public static Singleton getInstance() {
        if (sSingleton == null) {
            synchronized (Singleton.class) {
                // 未初始化，则初始instance变量
                if (sSingleton == null) {
                    sSingleton = new Singleton();
                }
            }
        }
        return sSingleton;
    }
}
```

再看看我们用 kotlin 实现

```
class Singleton private constructor(){
    companion object {
        @Volatile
        private var instance: Singleton? = null

        fun getInstance(context: Context): Singleton {
            return instance?: synchronized(this) {
                instance?:Singleton().also {
                    instance = it
                }
            }
        }
    }
}
```

如果要做初始化操作，我们完全可以在 also 函数里面去处理。