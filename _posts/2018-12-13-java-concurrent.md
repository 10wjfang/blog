---
layout: post
title: 异步编程：CompletableFuture
date: 2018-12-13 11:18:38
catalog: true
tags:
    - Java
---

## 并行与并发

**并行**：分支/合并框架以及并行流是实现并行处理的宝贵工具，它们将一个操作切分成多个子操作，在多个不同的核、CPU甚至是机器上并行地执行这些子操作。

**并发**：在同一个CPU上执行几个松耦合的任务，充分利用CPU的核，最大化程序的吞吐量，避免因为等待远程服务的返回，或者对数据库的查询，而阻塞线程的执行，浪费宝贵的计算资源。

并行与并发的区别：

![img](../../../../img/in-post/post-java/diff.png)

## CompletableFuture介绍

#### 1. 显式创建线程

```java
private static Future<Double> getPriceAsync() {
    CompletableFuture<Double> future = new CompletableFuture<>();
    new Thread(() -> {
        double price = calculatePrice();
        future.complete(price);
    }).start();
    return future;
}
```

当请求的产品价格最终计算得出时，你可以使用它的`complete`方法，结束`completableFuture`对象的运行，并设置变量的值。
再调用`Future`的`get`方法。执行了这个操作后，客户要么获得`Future`中封装的值（如果异步任务已经完成），要么发生阻塞，直到该异步任务完成，期望的值能够访问。

#### 2. 使用工厂方法创建

```java
private static Future<Double> getPriceAsync2() {
    return CompletableFuture.supplyAsync(() -> calculatePrice());
}
```

`supplyAsync`方法接受一个生产者（`Supplier`）作为参数，返回一个`CompletableFuture`对象，该对象完成异步执行后会读取调用生产者方法的返回值。

## 深入了解

本机CPU为8核，使用两个列表进行测试，一个有8个元素，另一个有9个元素。

8个元素的列表：
```java
List<Shop> shops = Arrays.asList(new Shop(), new Shop(), new Shop(), new Shop(), new Shop(),
                new Shop(), new Shop(), new Shop(), new Shop());
```

9个元素的列表：
```java
List<Shop> shops = Arrays.asList(new Shop(), new Shop(), new Shop(), new Shop(), new Shop(), new Shop(),
                new Shop(), new Shop(), new Shop(), new Shop());
```

每个`getPrice()`花费1s。

#### 1. 使用并行流

```java
private static List<String> findPrices(List<Shop> shops) {
    return shops.parallelStream()
            .map(shop -> String.format("price is %.2f", shop.getPrice()))
            .collect(Collectors.toList());
}
```

8个元素测试结果：
```
took = [1082]
price is 0.87, thread: ForkJoinPool.commonPool-worker-7
price is 0.06, thread: ForkJoinPool.commonPool-worker-3
price is 0.95, thread: ForkJoinPool.commonPool-worker-1
price is 0.25, thread: ForkJoinPool.commonPool-worker-5
price is 0.35, thread: ForkJoinPool.commonPool-worker-6
price is 0.45, thread: main
price is 0.97, thread: ForkJoinPool.commonPool-worker-4
price is 0.76, thread: ForkJoinPool.commonPool-worker-2
```

花费1.082s。

9个元素测试结果：
```
took = [2082]
price is 0.57, thread: ForkJoinPool.commonPool-worker-6
price is 0.18, thread: ForkJoinPool.commonPool-worker-3
price is 0.97, thread: ForkJoinPool.commonPool-worker-1
price is 0.97, thread: ForkJoinPool.commonPool-worker-5
price is 0.49, thread: ForkJoinPool.commonPool-worker-7
price is 0.37, thread: main
price is 0.73, thread: ForkJoinPool.commonPool-worker-4
price is 0.49, thread: ForkJoinPool.commonPool-worker-2
price is 0.83, thread: ForkJoinPool.commonPool-worker-7
```

花费2.082s。

#### 2. 使用CompletableFutrue

```java
private static List<String> findPricesAsync(List<Shop> shops) {
    ExecutorService executorService = new ThreadPoolExecutor(shops.size(), shops.size(),
            1, TimeUnit.MINUTES, new ArrayBlockingQueue<Runnable>(5), new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            // 使用守护进程，这种方式不会阻止程序的关停
            t.setDaemon(true);
            return t;
        }
    });
    List<CompletableFuture<String>> futures = shops.stream()
            .map(shop -> CompletableFuture.supplyAsync(
                    () -> String.format("price is %.2f, thread: %s",
                            shop.getPrice(), Thread.currentThread().getName()), executorService))
            .collect(Collectors.toList());
    // 等待所有异步操作结束
    return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
}
```

结果：
```
took = [1085]
price is 0.59, thread: Thread-0
price is 0.02, thread: Thread-1
price is 0.56, thread: Thread-2
price is 0.28, thread: Thread-3
price is 0.43, thread: Thread-4
price is 0.57, thread: Thread-5
price is 0.75, thread: Thread-6
price is 0.61, thread: Thread-7
price is 0.79, thread: Thread-8
```

花费1.085s。

这里使用了两个不同的Stream流水线，而不是在同一个处理流的流水线上一
个接一个地放置两个map操作。

并行流和CompletableFuture它们内部采用的是同样的通用线程池，默认都使用固定数目的线程，具体线程数取决于`Runtime.getRuntime().availableProcessors()`的返回值。然而， `CompletableFuture`具有一定的优势，因为它允许你对执行器（`Executor`）进行配置，调整线程池的大小。

> 线程池大小与处理器的利用率之比可以使用下面的公式进行估算：
Nthreads = NCPU * UCPU * (1 + W/C)
> - NCPU是处理器的核的数目，可以通过Runtime.getRuntime().availableProcessors()得到
> - UCPU是期望的CPU利用率（该值应该介于0和1之间）
> - W/C是等待时间与计算时间的比率

Java程序无法终止或者退出一个正在运行中的线程，所以最后剩下的那个线程会由于一直等待无法发生的事件而引发问题。与此相反，如果将线程标记为守护进程，意味着程序退出时它也会被回收。这二者之间没有性能上的差异。

## 并行——使用流还是CompletableFuture?

- 如果你进行的是计算密集型的操作，并且没有I/O，那么推荐使用Stream接口，因为实现简单，同时效率也可能是最高的（如果所有的线程都是计算密集型的，那就没有必要创建比处理器核数更多的线程）

- 反之，如果你并行的工作单元还涉及等待I/O的操作（包括网络连接等待），那么使用CompletableFuture灵活性更好，你可以像前文讨论的那样，依据等待/计算，或者W/C的比率设定需要使用的线程数。这种情况不使用并行流的另一个原因是，处理流的流水线中如果发生I/O等待，流的延迟特性会让我们很难判断到底什么时候触发了等待。

## 参考

《Java 8实战》