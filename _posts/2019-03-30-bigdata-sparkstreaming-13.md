---
layout: post
title: 【计算】实时流处理-Spark Streaming（十四）
date: 2019-3-29 15:35:03
catalog: true
tags:
    - 大数据
    - Spark Streaming
---

## 概述

`Spark Streaming`是核心Spark API的扩展，可实现可扩展、高吞吐量、可容错的实时数据流处理。数据可以从诸如`Kafka`，`Flume`，`Kinesis`或TCP套接字等众多来源获取，并且可以使用由高级函数（如map，reduce，join和window）开发的复杂算法进行流数据处理。最后，处理后的数据可以被推送到文件系统，数据库和实时仪表板。而且，您还可以在数据流上应用Spark提供的机器学习和图处理算法。

![img](../../../../img/in-post/post_bigdata/spark-streaming.png)

## 工作原理

它的工作原理如下：Spark Streaming接收实时输入数据流，并将数据切分成批，然后由Spark引擎对其进行处理，最后生成“批”形式的结果流。

![img](../../../../img/in-post/post_bigdata/spark-streaming2.png)

Spark Streaming将连续的数据流抽象为`discretizedstream`或`DStream`。 可以从诸如Kafka，Flume和Kinesis等来源的输入数据流中创建DStream，或者通过对其他DStream应用高级操作来创建。在内部，DStream 由一个`RDD`序列表示。

## 编写Spark Streaming程序

功能介绍：假设我们有一个数据服务器正在对一个TCP套接字进行侦听，然后需要统计接收的文本数据中的每个单词的出现频率。

添加依赖：

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.1.0</version>
</dependency>
```

代码实现：

```java
public class SparkTest {
    public static void main(String[] args) throws InterruptedException {
        if (args.length < 2) {
            System.err.println("Usage: JavaNetworkWordCount <hostname> <port>");
            System.exit(1);
        }
        // 1、创建JavaStreamingContext对象
        SparkConf conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount");
        JavaStreamingContext jssc = new JavaStreamingContext(conf, Durations.seconds(5));
        // 2、使用JavaStreamingContext创建DStream
        JavaReceiverInputDStream<String> lines = jssc.socketTextStream(args[0], Integer.parseInt(args[1]));
        // 3、拆分单词
        JavaDStream<String> words = lines.flatMap(x -> Arrays.asList(x.split(" ")).iterator());
        // 4、统计词频
        JavaPairDStream<String, Integer> pairs = words.mapToPair(s -> new Tuple2<>(s, 1));
        JavaPairDStream<String, Integer> wordCounts = pairs.reduceByKey((i1, i2) -> i1 + i2);
        wordCounts.print();
        // 5、启动
        jssc.start();
        jssc.awaitTermination();
    }
}
```

- `StreamingContext`是所有流功能的主要入口点。
- `DStream`表示从数据服务器接收的数据流。
- `appName`参数是应用程序在集群UI上显示的名称。
- `master`是Spark，Mesos或YARN集群的URL，或者一个特殊的“`local [*]`”字符串来让程序以本地模式运行。在具体的实践中，当您在集群上运行程序时，不需要在程序中硬编码master参数，而是使用`spark-submit`提交应用程序并将master的URL以脚本参数的形式传入。但是，对于本地测试和单元测试，您可以通过“`local[*]`”来运行Spark Streaming程序（请确保本地系统中的cpu核心数够用）。

## 测试

1、在运行spark程序之前您将首先需要运行Netcat（大多数类Unix系统中的一个小型实用程序）作为数据服务器。

```sh
nc -lk 9999
```

2、启动SparkDemo程序

3、在运行netcat服务器的终端中输入的任何行将每秒进行单词计数并打印在屏幕上。

```sh
$ nc -lk 9999
hello world
```

4、查看统计结果

```sh
···
(hello, 1)
(world, 1)
···
```

## 参考

* [Spark Streaming Programming Guide](http://spark.apache.org/docs/latest/streaming-programming-guide.html)
* [Spark2.1.0文档：Spark Streaming 编程指南（上）](https://blog.csdn.net/u013468917/article/details/71274433)