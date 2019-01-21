---
layout: post
title: ElasticSearch入门
date: 2019-1-21 14:59:20
catalog: true
tags:
    - ElasticSearch
---

## 安装ElasticSearch

1、下载ElasticSearch

下载地址：`https://www.elastic.co/cn/downloads/past-releases/elasticsearch-5-6-4`，上传到服务器，并解压到`/opt`目录。

2、修改配置文件

修改`elasticsearch/config/elasticsearch.yml`文件：

- 修改集群名称：`cluster.name: my-es`
- 修改数据存储目录：`path.data: /data/elastic/data`
- 修改日志存储目录：`path.log: /data/elastic/logs`
- 修改主机地址：`network.host: 398.cdh.slave2`
- 去掉注释：`http.port: 9200`
- 设置自动发现节点：`discovery.zen.ping.unicast.hosts: ["398.cdh.slave2"]`

## 启动ElasticSearch

*注意：需要先安装JDK，不能用root账号运行*

```sh
/opt/elasticsearch/bin/elasticsearch
```

**可能遇到问题：**

- 如果账号没有权限，`chown -R 用户名 /data/elastic`
- 提示`Ubuntu elasticsearch max virtual memory areas vm.max_map_count [65530] is too low`，解决：
    ```sh
    vi /etc/sysctl.conf
    vm.max_map_count=262144
    sysctl -p
    ```

## 测试

访问：`http://192.168.10.32:9200/_cat/health?v`

## 安装ik分词插件

```sh
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.6.4/elasticsearch-analysis-ik-5.6.4.zip
```