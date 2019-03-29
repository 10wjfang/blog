---
layout: post
title: 【计算】计算引擎-Spark（十三）
date: 2019-3-29 15:35:03
catalog: true
tags:
    - 大数据
    - Spark
---

## 什么是Spark

spark是一个实现快速通用的集群计算平台。它是由加州大学伯克利分校AMP实验室 开发的通用内存并行计算框架，用来构建大型的、低延迟的数据分析应用程序。它扩展了广泛使用的MapReduce计算模型。高效的支撑更多计算模式，包括交互式查询和流处理。spark的一个主要特点是能够在内存中进行计算，及时依赖磁盘进行复杂的运算，Spark依然比MapReduce更加高效。

## Spark的四大特性

- **高效性**：运行速度提高100倍。
- **易用性**：支持Java、Python和Scala的API，还支持超过80种高级算法。
- **通用性**：可以用于批处理、交互式查询（Spark SQL）、实时流处理（Spark Streaming）、机器学习（Spark MLlib）和图计算（GraphX）。
- **兼容性**：可以非常方便地与其他的开源产品进行融合，比如HDFS、HBase和Cassandra等。

## Spark的组成

- SparkCore：将分布式数据抽象为弹性分布式数据集（RDD），实现了应用任务调度、RPC、序列化和压缩，并为运行在其上的上层组件提供API。

- SparkSQL：Spark Sql 是Spark来操作结构化数据的程序包，可以让我使用SQL语句的方式来查询数据，Spark支持 多种数据源，包含Hive表，parquest以及JSON等内容。

- SparkStreaming： 是Spark提供的实时数据进行流式计算的组件。

- MLlib：提供常用机器学习算法的实现库。

- GraphX：提供一个分布式图计算框架，能高效进行图计算。

- BlinkDB：用于在海量数据上进行交互式SQL的近似查询引擎。

- Tachyon：以内存为中心高容错的的分布式文件系统。

## 安装Spark

通过CDH安装Spark。

## 编写Spark程序

1、登录到某一个节点上后，切换到`hdfs`用户：

```sh
su hdfs # root用户没有权限
cd # 不能在/root目录下操作
```

2、编写一个hello.txt文件并上传到HDFS上的spark目录下

```sh
hdfs@398:~$ hdfs dfs -mkdir -p /spark
hdfs@398:~$ hdfs dfd -put hello.txt /spark
```

hello.txt的内容如下：

```txt
you,jump
i,jump
you,jump
i,jump
jump
```

3、启动spark-shell

启动spark on yarn：

```sh
spark-shell --master yarn-client
```

![img](../../../../img/in-post/post_bigdata/spark-shell.png)

打开YARN WEB页面：

![img](../../../../img/in-post/post_bigdata/yarn.png)

3、在spark shell中用scala语言编写spark程序

```scala
scala> sc.textFile("/spark/hello.txt").flatMap(_.split(",")).map((_,1)).reduceByKey(_+_).saveAsTextFile("/spark/out")
```

- sc是`SparkContext`对象，该对象是提交spark程序的入口
- `textFile("/spark/hello.txt")`是hdfs中读取数据
- `flatMap(_.split(" "))`先map再压平
- `map((_,1))`将单词和1构成元组
- `reduceByKey(_+_)`按照key进行reduce，并将value累加
- `saveAsTextFile("/spark/out")`将结果写入到hdfs中

4、使用hdfs命令查看结果

```sh
hdfs@398:~$ hdfs dfs -cat /spark/out/*
(jump,5)
(you,2)
(i,2)
```

查看Spark任务：

![img](../../../../img/in-post/post_bigdata/spark-jobs.png)

## 执行Spark自带的示例程序PI

```sh
spark-submit --class org.apache.spark.examples.SparkPi --master yarn-client /opt/cloudera/parc
els/CDH-5.15.1-1.cdh5.15.1.p0.4/lib/spark/examples/lib/spark-examples-1.6.0-cdh5.15.1-hadoop2.6.0-cdh5.15.1.jar 10
```

![img](../../../../img/in-post/post_bigdata/spark-pi.png)

## 参考

* [官网地址](http://spark.apache.org/)
* [Spark学习之路 （二）Spark2.3 HA集群的分布式安装](https://www.cnblogs.com/qingyunzong/p/8888080.html)