---
layout: post
title: 关于synchronized和ReentrantLock之多线程同步
date: 2018-11-14 14:16:06
catalog: true
tags:
    - Java
    - 并发编程
---

## 问题

假如有个一买票系统，当前总共100张票，有4个窗口在卖。

```java
public class SynchronizeDemo implements Runnable {
    private int num = 100;

    @Override
    public void run() {
        while (num > 0) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " - num: " + num--);
        }
        System.out.println("票卖完了");
    }

    public static void main(String[] args) {
        SynchronizeDemo s = new SynchronizeDemo();
        Thread t1 = new Thread(s);
        Thread t2 = new Thread(s);
        Thread t3 = new Thread(s);
        Thread t4 = new Thread(s);
        t1.start();
        t2.start();
        t3.start();
        t4.start();
    }
}
```

结果出现票数为负数：

```
Thread-2 - num: 1
票卖完了
Thread-1 - num: 0
票卖完了
Thread-0 - num: -1
票卖完了
Thread-3 - num: -2
票卖完了
```

这就是在多线程的情况下，出现了数据的“脏读”。即多个线程访问余票时时，可能某个线程已经把票卖完，其它线程拿到的余票不是最新的，这样就出现了卖同一张票的情况。

## 解决

通过使用synchronized和ReentrantLock来实现线程的同步。

#### synchronized方式

**1. synchronized简介**

synchronized实现同步的基础：Java中每个对象都可以作为锁。当线程试图访问同步代码时，必须先获得对象锁，退出或抛出异常时必须释放锁。Synchronzied实现同步的表现形式分为：代码块同步和方法同步。

**2. synchronized原理**

代码块同步：在编译后通过将monitorenter指令插入到同步代码块的开始处，将monitorexit指令插入到方法结束处和异常处，通过反编译字节码可以观察到。任何一个对象都有一个monitor与之关联，线程执行monitorenter指令时，会尝试获取对象对应的monitor的所有权，即尝试获得对象的锁。

方法同步：synchronized方法在method_info结构有ACC_synchronized标记，线程执行时会识别该标记，获取对应的锁，实现方法同步。

两者虽然实现细节不同，但本质上都是对一个对象的监视器（monitor）的获取。任意一个对象都拥有自己的监视器，当同步代码块或同步方法时，执行方法的线程必须先获得该对象的监视器才能进入同步块或同步方法，没有获取到监视器的线程将会被阻塞，并进入同步队列，状态变为BLOCKED。当成功获取监视器的线程释放了锁后，会唤醒阻塞在同步队列的线程，使其重新尝试对监视器的获取。

**3. synchronized使用场景**

**实例方法同步：**
```java
public synchronized void method1
```

**代码块同步：**
```java
synchronized(this){ //TODO }
```

锁住的是该对象,类的其中一个实例，当该对象(仅仅是这`一个对象`)在不同线程中执行这个同步方法时，线程之间会形成互斥。达到同步效果，但如果不同线程同时对该类的不同对象执行这个同步方法时，则线程之间不会形成互斥，因为他们拥有的是不同的锁。

**静态方法同步：**
```java
public synchronized static void method2
```

**代码块同步：**
```java
synchronized(Test.class){ //TODO}
```

锁住的是该类，当所有该类的对象(`多个对象`)在不同线程中调用这个static同步方法时，线程之间会形成互斥，达到同步效果。

**代码块同步：**
```java
synchronized(o) {}
```

这里面的o可以是一个任何Object对象或数组，并不一定是它本身对象或者类，谁拥有o这个锁，谁就能够操作该块程序代码。

具体解决方法如下：

```java
public void run() {
    while (num > 0) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        synchronized (this) {
            if (num > 0)
                System.out.println(Thread.currentThread().getName() + " - num: " + num--);
        }
    }
    System.out.println("票卖完了");
}
```

#### ReentrantLock锁方式

**1.ReentrantLock简介**

ReentrantLock，一个可重入的互斥锁，它具有与使用synchronized方法和语句所访问的隐式监视器锁相同的一些基本行为和语义，但功能更强大。

**2. Lock接口**

在Java中锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源（但有的锁可以允许多个线程并发访问共享资源，比如读写锁，后面我们会分析）。在Lock接口出现之前，Java程序是靠synchronized关键字（后面分析）实现锁功能的，而JAVA SE5.0之后并发包中新增了Lock接口用来实现锁的功能，它提供了与synchronized关键字类似的同步功能，只是在使用时需要显式地获取和释放锁，缺点就是缺少像synchronized那样隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性，可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

**void lock()：** 执行此方法时, 如果锁处于空闲状态, 当前线程将获取到锁. 相反, 如果锁已经被其他线程持有, 将禁用当前线程, 直到当前线程获取到锁.

**void unlock()：** 执行此方法时, 当前线程将释放持有的锁. 锁只能由持有者释放, 如果线程并不持有锁, 却执行该方法, 可能导致异常的发生.

**3. 重入锁**

当一个线程得到一个对象后，再次请求该对象锁时是可以再次得到该对象的锁的。
具体概念就是：自己可以再次获取自己的内部锁。
Java里面内置锁(synchronized)和Lock(ReentrantLock)都是可重入的。

**4. synchronized和ReenTrantLock 的区别**

**① synchronized 依赖于 JVM 而 ReenTrantLock 依赖于 API**

synchronized 是依赖于 JVM 实现的，前面我们也讲到了 虚拟机团队在 JDK1.6 为 synchronized 关键字进行了很多优化，但是这些优化都是在虚拟机层面实现的，并没有直接暴露给我们。ReenTrantLock 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock 方法配合 try/finally 语句块来完成）。

**② ReenTrantLock 比 synchronized 增加了一些高级功能**

相比synchronized，ReenTrantLock增加了一些高级功能。主要来说主要有三点：①等待可中断；②可实现公平锁；③可实现选择性通知（锁可以绑定多个条件）

解决方法如下：

```java
public class SynchronizeDemo implements Runnable {
    private int num = 100;
    ReentrantLock lock = new ReentrantLock();
    @Override
    public void run() {
        while (num > 0) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            lock.lock();
            if (num > 0)
                System.out.println(Thread.currentThread().getName() + " - num: " + num--);
            lock.unlock();
        }
        System.out.println("票卖完了");
    }
}
```

#### AtomicInteger方式

**1. Atomic原子类介绍**

Atomic 翻译成中文是原子的意思。在化学上，我们知道原子是构成一般物质的最小单位，在化学反应中是不可分割的。在我们这里 Atomic 是指一个操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程干扰。

所以，所谓原子类说简单点就是具有原子/原子操作特征的类。

**2. AtomicInteger 的使用**

```java
public final int get() //获取当前的值
public final int getAndSet(int newValue)//获取当前的值，并设置新的值
public final int getAndIncrement()//获取当前的值，并自增
public final int getAndDecrement() //获取当前的值，并自减
public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）
public final void lazySet(int newValue)//最终设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

**3. AtomicInteger 类的原理**

AtomicInteger 类主要利用 CAS (compare and swap) + volatile 和 native 方法来保证原子操作，从而避免 synchronized 的高开销，执行效率大为提升。

CAS的原理是拿期望的值和原本的一个值作比较，如果相同则把内存中的值更新成新的值。UnSafe 类的 objectFieldOffset() 方法是一个本地方法，这个方法是用来拿到“原来的值”的内存地址，返回值是 valueOffset。另外 value 是一个volatile变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值。

```java
public class SynchronizeDemo implements Runnable {
    private AtomicInteger num = new AtomicInteger(100);

    @Override
    public void run() {
        while (num.intValue() > 0) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            if (num.intValue() > 0)
                System.out.println(Thread.currentThread().getName() + " - num: " + num.decrementAndGet());
        }
        System.out.println("票卖完了");
    }
}
```

## 参考

> 作者：Ruheng
链接：https://www.jianshu.com/p/96c89e6e7e90
來源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。

[BATJ都爱问的多线程面试题](https://github.com/Snailclimb/JavaGuide/blob/master/Java%E7%9B%B8%E5%85%B3/Multithread/BATJ%E9%83%BD%E7%88%B1%E9%97%AE%E7%9A%84%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%9D%A2%E8%AF%95%E9%A2%98.md)