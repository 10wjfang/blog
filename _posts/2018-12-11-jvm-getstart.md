---
layout: post
title: JVM介绍
date: 2018-12-11 21:57:51
catalog: true
tags:
    - Java
    - JVM
---

`JDK 1.6`

## JVM是什么？

JVM是Java Virtual Machine的缩写，JVM是一种用于计算设备的规范，它是一个虚构出来的计算机，是通过在实际的计算机上仿真模拟各种计算机功能来实现的。

## JVM的位置

![img](../../../../img/in-post/post-java/jdk.jpg)

## JVM运行时数据区

![img](../../../../img/in-post/post-java/jvm.jpg)

#### 程序计数器

**指向当前线程正在执行的字节码指令的`地址`（行号）。** Java最小的执行单位是线程，这是由于CPU有时间片的概念，线程在一个时间片执行不完就会挂起，等待获取CPU资源后再次执行，这是就需要从程序计数器里获取当前线程执行的地址继续执行。

#### 虚拟机栈

**存储当前`线程`运行`方法`时所需要的数据、指令、返回地址。**
每个方法被执行的时候都会同时创建一个`栈帧`，用于存储`局部变量表`、`操作数栈`、`动态链接`、`方法出口`等信息。

- **局部变量表**：存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用，32位的寻址空间。在编译期就确定大小。
- **操作数栈**：和局部变量区一样，操作数栈也是被组织成一个以字长为单位的数组。但是和前者不同的是，它不是通过索引来访问，而是通过标准的栈操作—压栈和出栈—来访问的。比如，如果某个指令把一个值压入到操作数栈中，稍后另一个指令就可以弹出这个值来使用。
- **动态链接**：每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接。
- **方法出口**：方法退出的过程实际上等同于把当前栈帧出栈，因此退出时可能执行的操作有：恢复上层方法的局部变量表和操作数栈，如果有返回值，则把它压入调用者栈帧的操作数栈中，调整PC计数器的值以指向方法调用指令后面的一条指令。

> 反编译：javap -v ClassName.class >> className.txt，可以生成字节码文件

栈内存在JVM中默认是1M，可以通过下面的参数进行设置

> -Xss

#### 本地方法栈

类似虚拟机栈，运行的是本地方法，也就是带native的方法。

#### 方法区

存储类信息、常量（1.7+有变化）、静态变量、JIT（1.7以前），1.7+字符串常量存到堆里。

#### 堆（Heap）

用来存储程序中的一些对象，比如你用new关键字创建的对象，它就会被存储在堆内存中，但是这个对象在堆内存中的首地址会存储在栈中。

![img](../../../../img/in-post/post-java/heap.jpg)

> 1.8+没有永久代，改成Meta Space，解决永久代溢出问题，可扩容。需要限制最大值。

1. JVM中共享数据空间可以分成三个大区，新生代（Young Generation）、老年代（Old Generation）、永久代（Permanent Generation），其中JVM堆分为新生代和老年代。

2. 新生代可以划分为三个区，Eden区（存放新生对象），两个幸存区（From Survivor和To Survivor）（存放每次垃圾回收后存活的对象）。

3. 永久代管理class文件、静态对象、属性等。

4. JVM垃圾回收机制采用“分代收集”：新生代采用复制算法，老年代采用标记清理算法。

默认Eden:S0:S1=8:1:1。

最小堆内存在JVM中默认物理内存的64分之1，最大堆内存在JVM中默认物理内存4分之一，且建议最大堆内存不大于4G，并且设置-Xms=-Xmx避免每次GC后，调整堆的大小，减少系统内存分配开销。

> -Xms 初始化内存

> -Xmx 最大堆内存

> -Xmn 年轻代大小

- 当年轻代需要回收时会触发Minor GC(也称作Young GC)。
- 当老年代满了的时候就需要对老年代进行垃圾回收，老年代的垃圾回收称作Full GC。老年代所占用的内存大小为-Xmx对应的值减去-Xmn对应的值。

## GC ROOT

垃圾回收判定对象是不是垃圾对象用的是可达性分析算法。其中可达性分析算法是从GC Root开始分析对象的可达性，即有没有被引用。GC可达的对象是不能被回收的，GC不可达的对象是可以被回收的。没有被GC Root引用就会被垃圾回收器标记为可回收垃圾。

从一个节点GC ROOT开始，寻找对应的引用节点，找到这个节点以后，继续寻找这个节点的引用节点，当所有的引用节点寻找完毕之后，剩余的节点则被认为是没有被引用到的节点，即无用的节点。
那怎么确定GC Root呢，一般包含以下几种对象：

1、虚拟机栈（栈中的`局部变量表`）中引用的对象；

2、方法区中类`静态`属性引用的对象；

3、方法区中`常量`引用的对象；

4、`本地方法栈`中JNI（即一般说的Native方法）引用的对象。

## 垃圾回收器

分代的垃圾回收策略，是基于这样一个事实：不同的对象的生命周期是不一样的。因此，不同生命周期的对象可以采取不同的回收算法，以便提高回收效率。

新生代收集器使用的收集器：Serial、PraNew、Parallel Scavenge
老年代收集器使用的收集器：Serial Old、Parallel Old、CMS

## GC日志

可以通过在java命令种加入参数来指定对应的gc类型，打印gc日志信息并输出至文件等策略。
```
-XX:+PrintGC 输出GC日志
-XX:+PrintGCDetails 输出GC的详细日志
-XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
-XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息
-Xloggc:../logs/gc.log 日志文件的输出路径
```

例如:eclipse.ini中配置下面代码启动后会在同一目录下生成gc.log

```
-Xloggc:gc.log
-XX:+PrintGCTimeStamps
-XX:+PrintGCDetails
```

对于新生代回收的一行日志，其基本内容如下：

```log
2014-07-18T16:02:17.606+0800: 611.633: [GC 611.633: [DefNew: 843458K->2K(948864K), 0.0059180 secs] 2186589K->1343132K(3057292K), 0.0059490 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

其含义大概如下：

2014-07-18T16:02:17.606+0800（当前时间戳）: 611.633（时间戳）: [GC（表示Young GC） 611.633: [DefNew（单线程Serial年轻代GC）: 843458K（年轻代垃圾回收前的大小）->2K（年轻代回收后的大小）(948864K（年轻代总大小）), 0.0059180 secs（本次回收的时间）] 2186589K（整个堆回收前的大小）->1343132K（整个堆回收后的大小）(3057292K（堆总大小）), 0.0059490 secs（回收时间）] [Times: user=0.00（用户耗时） sys=0.00（系统耗时）, real=0.00 secs（实际耗时）]

老年代回收的日志如下：

```
2014-07-18T16:19:16.794+0800: 1630.821: [GC 1630.821: [DefNew: 1005567K->111679K(1005568K), 0.9152360 secs]1631.736: [Tenured:
2573912K->1340650K(2574068K), 1.8511050 secs] 3122548K->1340650K(3579636K), [Perm : 17882K->17882K(21248K)], 2.7854350 secs] [Times: user=2.57 sys=0.22, real=2.79 secs]
```

## MAT分析dump文件

#### 内存泄漏

在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是可达的，即在有向图中，存在通路可以与其相连；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却`一直占用内存`。

#### 使用jmap工具生成dump文件

jmap是JDK自带的一种用于生成内存镜像文件的工具，通过该工具，开发人员可以快速生成dump文件。

常用命令：jmap -dump:`<dump-options><pid>`

例如：jmap -dump:format=b,file=20170307.dump 16048

#### MAT工具的下载和安装

MAT(Memory Analyzer Tool)工具是eclipse的一个插件，使用起来非常方便，尤其是在分析大内存的dump文件时，可以非常直观的看到各个对象在堆空间中所占用的内存大小、类实例数量、对象引用关系、利用OQL对象查询，以及可以很方便的找出对象GC Roots的相关信息，当然最吸引人的还是能够快速为开发人员生成内存泄露报表，方便定位问题和分析问题。

MAT工具的下载地址为：http://www.eclipse.org/mat/downloads.php

MAT插件的下载地址为：http://download.eclipse.org/mat/1.3/update-site/

#### 使用MAT工具进行内存泄露分析

选项“-XX:+HeapDumpOnOutOfMemoryError ”和-“XX:HeapDumpPath”所代表的含义就是当程序出现OutofMemory时，将会在相应的目录下生成一份dump文件，而如果不指定选项“XX:HeapDumpPath”则在当前目录下生成dump文件。

在Leak Suspects页面会给出可能的内存泄露。


## 常见配置汇总

1.堆设置
- -Xms:初始堆大小
- -Xmx:最大堆大小
- -XX:NewSize=n:设置年轻代大小
- -XX:NewRatio=n:设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
- -XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
- -XX:MaxPermSize=n:设置持久代大小

2.收集器设置
- -XX:+UseSerialGC:设置串行收集器
- -XX:+UseParallelGC:设置并行收集器
- -XX:+UseParalledlOldGC:设置并行年老代收集器
- -XX:+UseConcMarkSweepGC:设置并发收集器

3.垃圾回收统计信息
- -XX:+PrintGC
- -XX:+PrintGCDetails
- -XX:+PrintGCTimeStamps
- -Xloggc:filename

4.并行收集器设置
- -XX:ParallelGCThreads=n:设置并行收集器收集时使用的CPU数。并行收集线程数。
- -XX:MaxGCPauseMillis=n:设置并行收集最大暂停时间
- -XX:GCTimeRatio=n:设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n)

5.并发收集器设置
- -XX:+CMSIncrementalMode:设置为增量模式。适用于单CPU情况。
- -XX:ParallelGCThreads=n:设置并发收集器年轻代收集方式为并行收集时，使用的CPU数。并行收集线程数。

## 参考

[java中JVM的原理重温【转】](https://www.cnblogs.com/eastday/p/8124580.html)

[JVM简介](https://www.cnblogs.com/of-fanruice/p/7683308.html)

[JVM调优总结 -Xms -Xmx -Xmn -Xss](https://www.cnblogs.com/lcword/p/5857918.html)

[利用内存分析工具（Memory Analyzer Tool，MAT）分析java项目内存泄露](https://blog.csdn.net/wanghuiqi2008/article/details/50724676)