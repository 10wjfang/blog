---
layout: post
title: SpringBoot整合Kafka（二）
date: 2019-3-28 16:38:37
catalog: true
tags:
    - Kafka
    - Spring Boot
---

## 环境准备

- 启动zk，kafka_1.0.1
- 创建一个Topic

## 具体实现

1、添加pom依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

2、修改配置文件

```properties
spring.kafka.bootstrap-servers=192.168.241.140:9092,192.168.241.141:9092,192.168.241.142:9092
spring.kafka.consumer.group-id=test-consumer-group
```

- `consumer.group-id`必须配置，可以看kafka的`consumer.propertis`

3、新建生产者类

```java
@RestController
public class KafkaProducerController {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @RequestMapping("send")
    public String send(String msg) {
        kafkaTemplate.send("topic_1", msg);
        return "success";
    }
}
```

4、新建消费者类

```java
@Component
public class KafkaConsumerController {
    @KafkaListener(topics = "topic_1")
    public void listen(ConsumerRecord<?, ?> record) throws Exception {
        System.out.printf("topic = %s, offset = %d, value = %s \n", record.topic(), record.offset(), record.value());
    }
}
```

## 测试

运行项目，
在浏览器输入：`http://localhost:8080/send?msg=hello22`，正常可以看到控制台输出：

```sh
topic = topic_1, offset = 3, value = hellofang 
topic = topic_1, offset = 4, value = hello22 
```

## FAQ

- 启动提示`Marking the coordinator localhost:9092`：在hosts文件添加主机名IP映射。

## 参考

* [SpringBoot（十二）：SpringBoot整合Kafka](https://blog.csdn.net/saytime/article/details/79950635)