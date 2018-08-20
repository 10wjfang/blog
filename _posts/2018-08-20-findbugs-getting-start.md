---
layout: post
title: 关于代码检查工具Findbugs的运用
date: 2018-8-20 10:52:37
catalog: true
tags:
    - 工具
---

## 介绍

Findbugs是一个静态分析工具,它检查类或者jar文件,将字节码和一组缺陷模式进行对比以发现可能的问题。有了静态分析工具,就可以在不实际运行程序的情况下对软件进行分析。不是通过分析类文件的形式或结构来确定程序的意图,而是通常使用Visitor模式来鉴别代码是否符合一些固定的规范。Findbugs可作为一款插件用在Eclipse或 IntelliJ IDEA环境的编译器上。

## 错误类型说明

#### Bad practice 坏的实践

下面列举几个：
HE：类中equals()与hashCode()没有同时定义，或者使用了错误的对象的hashCode()或equals()。
SQL：Statement 的execute方法调用了非常量的字符串；或Prepared Statement是由一个非常量的字符串产生。
DE： 方法终止或不处理异常，一般情况下，异常应该被处理或报告，或被方法抛出。

#### Correctness 一般的正确性问题

可能导致错误的代码，下面列举几个：
NP： 空指针被引用；在方法的异常路径里，空指针被引用；方法没有检查参数是否null；null值产生并被引用；null值产生并在方法的异常路径被引用；传给方法一个声明为@NonNull的null参数；方法的返回值声明为@NonNull实际是null。
Nm： 类定义了hashcode()方法，但实际上并未覆盖父类Object的hashCode()；类定义了tostring()方法，但实际上并未覆盖父类Object的toString()；很明显的方法和构造器混淆；方法名容易混淆。
SQL：方法尝试访问一个Prepared Statement的0索引；方法尝试访问一个ResultSet的0索引。
UwF：所有的write都把属性置成null，这样所有的读取都是null，这样这个属性是否有必要存在；或属性从没有被write。

#### Internationalization 国际化

当对字符串使用upper或lowercase方法，如果是国际的字符串，可能会不恰当的转换。

#### Malicious code vulnerability 可能受到的恶意攻击

如果代码公开，可能受到恶意攻击的代码，下面列举几个：
FI： 一个类的finalize()应该是protected，而不是public的。
MS：属性是可变的数组；属性是可变的Hashtable；属性应该是package protected的。

#### Multithreaded correctness 多线程的正确性

多线程编程时，可能导致错误的代码，下面列举几个：
ESync：空的同步块，很难被正确使用。
MWN：错误使用notify()，可能导致IllegalMonitorStateException异常；或错误的使用wait()。
No： 使用notify()而不是notifyAll()，只是唤醒一个线程而不是所有等待的线程。
SC： 构造器调用了Thread.start()，当该类被继承可能会导致错误。

#### Performance 性能问题

可能导致性能不佳的代码，下面列举几个：
DM：方法调用了低效的Boolean的构造器，而应该用Boolean.valueOf(…)；用类似Integer.toString(1) 代替new Integer(1).toString()；方法调用了低效的float的构造器，应该用静态的valueOf方法。
SIC：如果一个内部类想在更广泛的地方被引用，它应该声明为static。
SS： 如果一个实例属性不被读取，考虑声明为static。
UrF：如果一个属性从没有被read，考虑从类中去掉。
UuF：如果一个属性从没有被使用，考虑从类中去掉。

#### Dodgy 危险的，具有潜在危险的代码

可能运行期产生错误，下面列举几个：
CI： 类声明为final但声明了protected的属性。
DLS：对一个本地变量赋值，但却没有读取该本地变量；本地变量赋值成null，却没有读取该本地变量。
ICAST： 整型数字相乘结果转化为长整型数字，应该将整型先转化为长整型数字再相乘。
INT：没必要的整型数字比较，如X <= Integer.MAX_VALUE。
NP： 对readline()的直接引用，而没有判断是否null；对方法调用的直接引用，而方法可能返回null。
REC：直接捕获Exception，而实际上可能是RuntimeException。
ST： 从实例方法里直接修改类变量，即static属性。

## IDEA安装FindBugs插件

1、 安装插件

打开File -> Setings -> Plugins -> Browse Repositories，搜索findbugs，安装FindBugs-IDEA

2、重启激活插件

3、使用

点击项目右键，FindBugs -> Analyze Project Files（整个工程），分析结束后会出现结果面板，点击对应的item在右边会定位到具体的代码，其中Correctness这个错误使我们重点关注的对象，根据提示进行处理。

## 参考

[FindBugs Bug说明](http://findbugs.sourceforge.net/bugDescriptions.html)

[FindBugs-IDEA插件的使用](https://blog.csdn.net/feibendexiaoma/article/details/72821781)