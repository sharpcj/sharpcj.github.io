---
layout: post
title: "Kotlin return@xxx 的坑"
date:  2023-08-13 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin"]
---

Kotlin Return 到标签

先看例子：

```
(1..5).forEach {
    if (it == 3) {
        return@forEach
    }
    println(it)
}
println("test over")
```

这段代码执行结果是什么？

错误：

```
1
2
test over
```

这里可千万不要想当然，认为循环会结束。正确的结果是

```
1
2
4
5
test over
```

关于 Kotlin 中 return 使用的官方文档：
[https://www.kotlincn.net/docs/reference/returns.html](https://www.kotlincn.net/docs/reference/returns.html)

先说结论：

**1. return 默认从最直接包围它的函数或者匿名函数返回。**
**2. return 后面跟标签，返回到标签。**

关于第一点，如下代码：

```
fun test() {
    (1..5).forEach {
        if (it == 3) {
            return
        }
        println(it)
    }
    println("test over")
}
``

此时 return 会直接从 test() 函数中返回，输出：

```
1
2
```

对于第二点，返回到标签，首先名明白什么是标签。在 Kotlin 中任何表达式都可以用标签来标记，标签的格式为标识符后面跟@ 符号，例如 `abc@`、`aaa@` 等都是有效标识符。

再看开头的例子，为什么没有跳出循环？

其实例子中 `@forEach` 是一个和隐式标签，该例子等价于：

```
(1..5).forEach xxx@{
    if (it == 3) {
        return@xxx // 返回到 lambda 表达式的调用者
    }
    println(it)
}
println("test over")
```

forEach 后面跟的式一个lambda 表达式，此时的 return 实际上是返回到该 lambda 表达式的调用者，而并跳出 forEach 循环。

等价于下面使用匿名函数的实现方式，代码如下：

```
(1..5).forEach(fun(value: Int) {
    if (value == 3) return // 返回到匿名函数的调用者
    println(value)
})
println("test over")
```

那如果想要在循环到 3 的时候跳出循环，而不跳出函数，该如何办呢？

我们需要再调用 forEach 循环的调用处添加一个标签，再返回到该标签即可。如下：

```
run www@{
    (1..5).forEach {
        if (it == 3) {
            return@www
        }
        println(it)
    }
    println("test over")
}
```

知识点虽小，但是本人真真切切地踩过坑。记录下来，也希望其他人忽视了这个知识点的人，看到以后不要踩坑了。。。