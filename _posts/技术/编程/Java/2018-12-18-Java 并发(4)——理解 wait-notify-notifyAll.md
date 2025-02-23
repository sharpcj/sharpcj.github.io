---
layout: post
title: "Java 并发(4)———理解 wait-notify-notifyAll"
date:  2018-12-18 17:58:12 +0800
categories: ["技术", "编程", "Java"]
tag: ["Java", "并发", "wait-notify-notifyAll"]
---

## 一、前言
java 面试是否有被问到过，`sleep` 和 `wait` 方法的区别，关于这个问题其实不用多说，大多数人都能回答出最主要的两点区别：

- sleep 是线程的方法， wait / notify / notifyAll 是 Object 类的方法；
- sleep 不会释放当前线程持有的锁，到时间后程序会继续执行，`wait` 会释放线程持有的锁并挂起，直到通过 `notify` 或者 `notifyAll` 重新获得锁。
另外还有一些参数、异常等区别，不细说了。本文重点记录一下 wait / notify / notifyAll 的相关知识。

## 二、常见的同步场景
开发中常常遇到这样的场景：

```
一个线程执行过程中，需要开启另外一个子线程去做某个耗时的操作（通过休眠3秒模拟），
并且**等待**子线程返回结果，主线程再根据返回的结果继续往下执行。
```

这里注意我上面加*两个字“等待”。如果不需要等待，单纯只是对子线程的结果做处理，我们大可注册回调方法解决问题，此文不再赘述接口回调。
此处场景就是主线程停下来等待子线程执行完毕后，主线程再继续执行。针对该场景下面给出实现：

### 设置一个判断的标志位

```
volatile boolean flag = false;

public void test(){
    //...

    Thread t1 = new Thread(() -> {
        try {
            Thread.sleep(3000);
            System.out.println("--- 休眠 3 秒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            flag = true;
        }
    });
    t1.start();

    while(!flag){

    }
    System.out.println("--- work thread run");
}
```

上面的代码，执行结果：

![](/assets/images/技术/编程/java/Java并发(4)——理解%20wait-notify-notifyAll/pic1.gif)

强调一点，声明标志位的时候，一定注意 volatile 关键字不能忘，如果不加该关键字修饰，程序可能进入死循环。这是同步中的可见性问题，在 《java 并发——内置锁》 中有记录。
显然，这个实现方案并不好，本来主线程什么也不用做，却一直在竞争资源，做空循环，性能上不好，所以并不推荐。

### 线程的 join 方法

```
public void test(){
    //...

    Thread t1 = new Thread(() -> {
        try {
            Thread.sleep(3000);
            System.out.println("--- 休眠 3 秒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });

    t1.start();

    try {
        t1.join();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("--- work thread continue");
}
```

上面的代码，执行结果同上。利用 Thread 类的 join 方法实现了同步，达到了效果，但是 join 方法不能一定保证效果，在不同的 cpu 上，可能呈现出意想不到的结果，所以尽量不要用上述方法。

### 使用闭锁 CountDownLatch
不清楚闭锁的新同学可点击文章开头给出的另一篇文章，《java 并发——线程》。

```
public void test(){
    //...

    final CountDownLatch countDownLatch = new CountDownLatch(1);

    Thread t1 = new Thread(() -> {
        try {
            Thread.sleep(3000);
            System.out.println("--- 休眠 3 秒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            countDownLatch.countDown();
        }
    });

    t1.start();

    try {
        countDownLatch.await();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("--- work thread run");
}
```

上面的代码，执行结果同上。同样可以实现上述效果，执行结果和上面一样。该方法推荐使用。

### 利用 wait / notify 优化标志位方法
为了方便对比，首先给 2.1 中的循环方法增加一些打印。修改后的代码如下：

```
volatile boolean flag = false;

public void test() {
    //...
    Thread t1 = new Thread(() -> {
        try {
            Thread.sleep(3000);
            System.out.println("--- 休眠 3 秒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            flag = true;
        }
    });
    t1.start();

    while (!flag) {
        try {
            System.out.println("---while-loop---");
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    System.out.println("--- work thread run");
}
```
执行结果如下：

![](/assets/images/技术/编程/java/Java并发(4)——理解%20wait-notify-notifyAll/pic2.gif)

事实证明，while 循环确实一直在执行。

为了使该线程再不需要执行的时候不抢占资源，我们可以利用 wait 方法将其挂起，在需要它执行的时候，再利用 notify 方法将其唤醒。这样达到优化的目的，优化后的代码如下：

```
volatile boolean flag = false;

public void test() {
    //...
    final Object obj = new Object();
    Thread t1 = new Thread(() -> {
        synchronized (obj) {
            try {
                Thread.sleep(3000);
                System.out.println("--- 休眠 3 秒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                flag = true;
            }
            obj.notify();
        }
    });
    t1.start();

    synchronized (obj) {
        while (!flag) {
            try {
                System.out.println("---while-loop---");
                Thread.sleep(500);
                obj.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    System.out.println("--- work thread run");
}
```

执行结果：

![](/assets/images/技术/编程/java/Java并发(4)——理解%20wait-notify-notifyAll/pic3.gif)

结果证明，优化后的程序，循环只执行了一次。

## 三、理解 wait / notify / notifyAll
在Java中，每个对象都有两个池，锁(monitor)池和等待池

### 锁池
锁池:假设线程A已经拥有了某个对象的锁，而其它的线程想要调用这个对象的某个synchronized方法(或者synchronized块)，由于这些线程在进入对象的synchronized方法之前必须先获得该对象的锁的拥有权，但是该对象的锁目前正被线程A拥有，所以这些线程就进入了该对象的锁池中。

### 等待池
等待池:假设一个线程A调用了某个对象的wait()方法，线程A就会释放该对象的锁(因为wait()方法必须出现在synchronized中，这样自然在执行wait()方法之前线程A就已经拥有了该对象的锁)，同时线程A就进入到了该对象的等待池中。如果另外的一个线程调用了相同对象的notifyAll()方法，那么处于该对象的等待池中的线程就会全部进入该对象的锁池中，准备争夺锁的拥有权。如果另外的一个线程调用了相同对象的notify()方法，那么仅仅有一个处于该对象的等待池中的线程(随机)会进入该对象的锁池.

### notify 和 notifyAll 的区别
**wait()**
public final void wait() throws InterruptedException,IllegalMonitorStateException
该方法用来将当前线程置入休眠状态，直到接到通知或被中断为止。在调用 wait()之前，线程必须要获得该对象的对象级别锁，即只能在同步方法或同步块中调用 wait()方法。进入 wait()方法后，当前线程释放锁。在从 wait()返回前，线程与其他线程竞争重新获得锁。如果调用 wait()时，没有持有适当的锁，则抛出 IllegalMonitorStateException，它是 RuntimeException 的一个子类，因此，不需要 try-catch 结

**notify()**
public final native void notify() throws IllegalMonitorStateException
该方法也要在同步方法或同步块中调用，即在调用前，线程也必须要获得该对象的对象级别锁，的如果调用 notify()时没有持有适当的锁，也会抛出 IllegalMonitorStateException。
该方法用来通知那些可能等待该对象的对象锁的其他线程。如果有多个线程等待，则线程规划器任意挑选出其中一个 wait()状态的线程来发出通知，并使它等待获取该对象的对象锁（notify 后，当前线程不会马上释放该对象锁，wait 所在的线程并不能马上获取该对象锁，要等到程序退出 synchronized 代码块后，当前线程才会释放锁，wait所在的线程也才可以获取该对象锁），但不惊动其他同样在等待被该对象notify的线程们。当第一个获得了该对象锁的 wait 线程运行完毕以后，它会释放掉该对象锁，此时如果该对象没有再次使用 notify 语句，则即便该对象已经空闲，其他 wait 状态等待的线程由于没有得到该对象的通知，会继续阻塞在 wait 状态，直到这个对象发出一个 notify 或 notifyAll。这里需要注意：它们等待的是被 notify 或 notifyAll，而不是锁。这与下面的 notifyAll()方法执行后的情况不同。

**notifyAll()**
public final native void notifyAll() throws IllegalMonitorStateException
该方法与 notify ()方法的工作方式相同，重要的一点差异是：
notifyAll 使所有原来在该对象上 wait 的线程统统退出 wait 的状态（即全部被唤醒，不再等待 notify 或 notifyAll，但由于此时还没有获取到该对象锁，因此还不能继续往下执行），变成等待获取该对象上的锁，一旦该对象锁被释放（notifyAll 线程退出调用了 notifyAll 的 synchronized 代码块的时候），他们就会去竞争。如果其中一个线程获得了该对象锁，它就会继续往下执行，在它退出 synchronized 代码块，释放锁后，其他的已经被唤醒的线程将会继续竞争获取该锁，一直进行下去，直到所有被唤醒的线程都执行完毕。

## 四、生产者与消费者模式
生产者与消费者问题是并发编程里面的经典问题。接下来说说利用wait()和notify()来实现生产者和消费者并发问题：
显然要保证生产者和消费者并发运行不出乱，主要要解决：当生产者线程的缓存区为满的时候，就应该调用wait()来停止生产者继续生产，而当生产者满的缓冲区被消费者消费掉一块时，则应该调用notify()唤醒生产者，通知他可以继续生产；同样，对于消费者，当消费者线程的缓存区为空的时候，就应该调用wait()停掉消费者线程继续消费，而当生产者又生产了一个时就应该调用notify()来唤醒消费者线程通知他可以继续消费了。
下面是一个简单的代码实现：

```
package com.sharpcj;

import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Test {
    public static void main(String[] args) {
        Reposity reposity = new Reposity(600);
        ExecutorService threadPool = Executors.newCachedThreadPool();
        for(int i = 0; i < 10; i++){
            threadPool.submit(new Producer(reposity));
        }

        for(int i = 0; i < 10; i++){
            threadPool.submit(new Consumer(reposity));
        }
        threadPool.shutdown();
    }
}


class Reposity {
    private static final int MAX_NUM = 2000;
    private int currentNum;

    private final Object obj = new Object();

    public Reposity(int currentNum) {
        this.currentNum = currentNum;
    }

    public void in(int inNum) {
        synchronized (obj) {
            while (currentNum + inNum > MAX_NUM) {
                try {
                    System.out.println("入货量 " + inNum + " 线程 " + Thread.currentThread().getId() + "被挂起...");
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            currentNum += inNum;
            System.out.println("线程： " + Thread.currentThread().getId() + ",入货：inNum = [" + inNum + "], currentNum = [" + currentNum + "]");
            obj.notifyAll();
        }
    }

    public void out(int outNum) {
        synchronized (obj) {
            while (currentNum < outNum) {
                try {
                    System.out.println("出货量 " + outNum + " 线程 " + Thread.currentThread().getId() + "被挂起...");
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            currentNum -= outNum;
            System.out.println("线程： " + Thread.currentThread().getId() + ",出货：outNum = [" + outNum + "], currentNum = [" + currentNum + "]");
            obj.notifyAll();
        }
    }
}

class Producer implements Runnable {
    private Reposity reposity;

    public Producer(Reposity reposity) {
        this.reposity = reposity;
    }

    @Override
    public void run() {
        reposity.in(200);
    }
}

class Consumer implements Runnable {
    private Reposity reposity;

    public Consumer(Reposity reposity) {
        this.reposity = reposity;
    }

    @Override
    public void run() {
        reposity.out(200);
    }
}
```

执行结果：

![](/assets/images/技术/编程/java/Java并发(4)——理解%20wait-notify-notifyAll/pic4.gif)

## 五、写在后面
最后做几点总结：

1. 调用wait方法和notify、notifyAll方法前必须获得对象锁，也就是必须写在synchronized(锁对象){......}代码块中。

2. 当线程调用了wait方法后就释放了对象锁，否则其他线程无法获得对象锁。

3. 当调用 wait() 方法后，线程必须再次获得对象锁后才能继续执行。

4. 如果另外两个线程都在 wait，则正在执行的线程调用notify方法只能唤醒一个正在wait的线程（公平竞争，由JVM决定）。

5. 当使用notifyAll方法后，所有wait状态的线程都会被唤醒，但是只有一个线程能获得锁对象，必须执行完while(condition){this.wait();}后才释放对象锁。其余的需要等待该获得对象锁的线程执行完释放对象锁后才能继续执行。

6. 当某个线程调用notifyAll方法后，虽然其他线程被唤醒了，但是该线程依然持有着对象锁，必须等该同步代码块执行完（右大括号结束）后才算正式释放了锁对象，另外两个线程才有机会执行。

7. 第5点中说明， wait 方法的调用前的条件判断需放在循环中，否则可能出现逻辑错误。另外，根据程序逻辑合理使用 wait 即 notify 方法，避免如先执行 notify ，后执行 wait 方法，线程一直挂起之类的错误。