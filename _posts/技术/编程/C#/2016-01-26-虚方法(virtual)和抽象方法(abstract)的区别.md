---
layout: post
title: "虚方法(virtual)和抽象方法(abstract)的区别"
date:  2016-01-26 17:58:12 +0800
categories: ["技术", "编程", "C#"]
tag: ["C#", "面向对象"]
---

注：本文转载自 [https://www.cnblogs.com/michaelxu/archive/2008/04/01/1132633.html](https://www.cnblogs.com/michaelxu/archive/2008/04/01/1132633.html)

虚方法和抽象方法都可以供派生类重写，它们之间有什么区别呢？

1. 虚方法必须有实现部分，抽象方法没有提供实现部分，抽象方法是一种强制派生类覆盖的方法，否则派生类将不能被实例化。如：
```
//抽象方法
public abstract class Animal
{
    public abstract void Sleep();
    public abstract void Eat();
}

//虚方法
public class Animal
{
    public virtual void Sleep(){}
    public virtual void Eat(){}
}
```

2. 抽象方法只能在抽象类中声明，虚方法不是。其实如果类包含抽象方法，那么该类也是抽象的，也必须声明为抽象的。如：
```
public class Animal
{
    public abstract void Sleep();
    public abstract void Eat();
}
```
编译器会报错：
```
Main.cs(10): 'VSTest.Animal.Sleep()' is abstract but it is contained in nonabstract class 'VSTest.Animal'
Main.cs(11): 'VSTest.Animal.Eat()' is abstract but it is contained in nonabstract class 'VSTest.Animal'
```

3. 抽象方法必须在派生类中重写，这一点跟接口类似，虚方法不必。如：
```
public abstract class Animal
{
    public abstract void Sleep();
    public abstract void Eat();
}

public class Cat : Animal
{
    public override void Sleep()
    {
        Console.WriteLine( "Cat is sleeping" );
    }
    // we need implement Animal.Eat() here

}
```