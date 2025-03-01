---
layout: post
title: "Kotlin 函数与 lambda 表达式"
date:  2022-01-14 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin", "lambda 表达式"]
---


## 一、函数

代码块函数体：

```
fun sum(x: Int, y: Int): Int {
    return x + y
}
```

表达式函数体：

```
fun sum(x: Int, y: Int) = x + y
```

使用表达式函数体，一般情况下可以不声明返回值类型。在一些诸如递归等复杂情况下，即使是使用表达式函数体，也必须显示声明返回值类型。

>总结：
1. 函数参数必须显示声明类型
2. 非表达式函数体，函数参数必须显示声明类型， 返回值除了类型是 Unit，可以省略，其它情况，返回值都必须显示声明类型
3. 如果它是一个递归的函数，必须显示声明参数和返回值类型
4. 如果它是一个公有方法，为了代码的可读性，尽量显示声明函数参数和返回值类型

## 二、函数类型
Kotlin 中，函数类型的格式如下：

```
() -> Unit
```

>Kotlin 中，函数类型声明必须遵循以下几点：
1. 通过 -> 符号来组织参数类型和返回值类型。左边是参数类型，右边是返回值类型
2. 必须用一个括号来包裹参数类型
3. 返回值类型即使是 Unit, 也必须显示声明。

使用`::`对某个类的方法进行引用，比如：

```
class A {
    fun test()
}
```

我们可以使用如下方式保持对 test() 方法的引用

```
val a = A () // 创建类的对象
val f = a::test // 通过对象引用方法
f.invike() // 调用方法
```

## 三、lambda 表达式

```
val sum: (Int, Int) -> Int = {x: Int, y: Int -> x + y}
```

由于支持类型推导，可以简化为

```
val sum: (Int, Int) -> Int = {x, y -> x + y}
```

或者：

```
val sum = {x: Int, y: Int -> x + y}
```

>lambda 表达式语法：
1. lambda 表达式必须通过 {} 来包裹
2. 如果 lambda 声明了参数部分的类型，且返回值支持类型推导，则 lambda 表达式变量就可以省略函数类型声明
3. 如果 lambda 变量声明了函数类型，那么 lambda 表达式的参数部分的类型就可以省略
4. 如果 lambda 表达式返回的不是 Unit 类型，则默认最后一行表达式的值类型就是返回值类型。

函数 和 lambda表达式
1. fun 在没有等号、只有花括号的情况下，就是代码块函数体，如果返回值非 Unit，必须带 return

```
fun foo(x: Int) {
    print(x)
}

fun foo(x: Int, y: Int): Int {
    return x + y
}
```

2. fun 带有等号，没有花括号，是单表达式函数体，可以省略 return

```
fun foo(x: Int, y: Int) = x + y
```

3. 不管是用 val 还是 fun 声明，如果是等号加花括号的语法，就是声明一个 lambda 表达式。

```
val foo = { x: Int, y: Int ->
    x + y
}
// 调用方式： foo.invoke(1, 2) 或者 foo(1, 2)
```

```
fun foo(x: Int) = { y: Int ->
    x + y
}
// 调用方式： foo(1).invoke(2) 或者 foo(1)(2)
```

4. lambda 表达式自调用

```
{x: Int, y: Int -> x + y}(1, 2)
```
