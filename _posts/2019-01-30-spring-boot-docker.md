---
layout: post
title: 使用Docker部署Spring Boot
date: 2019-1-30 14:00:08
catalog: true
tags:
    - Spring Boot
    - Docker
---

## 在Windows安装Docker

1、下载地址：`http://mirrors.aliyun.com/docker-toolbox/windows/docker-toolbox/`

2、下载完成之后直接点击安装，安装成功后，桌边会出现三个图标。

3、点击 Docker QuickStart 图标来启动 Docker Toolbox 终端。
启动的时候会下载`boot2docker.ios`，如果比较慢，可以复制链接下载后放到指定路径，再点击 Docker QuickStart 图标启动。

#### 修改镜像地址

1、Docker 官方中国区
https://registry.docker-cn.com

2、网易
http://hub-mirror.c.163.com

3、ustc
https://docker.mirrors.ustc.edu.cn

对于已创建的Docker Machine实例，更换镜像源的方法如下：
1、在Windows命令行执行`docker-machine ssh [machine-name]`进入VM bash，默认是`default`

2、`sudo vi /var/lib/boot2docker/profile`

3、在`--label provider=virtualbox`的下一行添加`--registry-mirror https://registry.docker-cn.com`

4、重启docker服务：`sudo /etc/init.d/docker restart`或者重启VM：`exit`退出VM bash，在Windows命令行中执行`docker-machine restart`

## 修改Spring Boot项目pom.xml

`docker-maven-plugin`和`dockerfile-maven-plugin`都可以用来构建镜像，是由同一个作者创造，作者明确表示推荐使用`dockerfile-maven-plugin`，并会持续升级；而`docker-maven-plugin`不在添加任何新功能，只接受修复bug。
推荐使用maven插件：`dockerfile-maven-plugin`。

1、添加镜像名称

```xml
<properties>
    <docker.image.prefix>398bigdata</docker.image.prefix>
</properties>
```

2、plugins 中添加 `Dockerfile` 构建插件：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>dockerfile-maven-plugin</artifactId>
            <configuration>
                <repository>${docker.image.prefix}/${project.artifactId}</repository>
                <tag>${project.version}</tag>
                <buildArgs>
                    <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                </buildArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 使用Docker部署

1、在项目根目录下添加`Dockerfile`文件

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG JAR_FILE
ADD ${JAR_FILE} app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

这个 Dockerfile 文件很简单，构建 Jdk 基础环境，添加 Spring Boot Jar 到镜像中，简单解释一下:

- FROM ，表示使用 Jdk8 环境 为基础镜像，如果镜像不是本地的会从 DockerHub 进行下载
- VOLUME ，VOLUME 指向了一个`/tmp`的目录，由于 Spring Boot 使用内置的Tomcat容器，Tomcat 默认使用`/tmp`作为工作目录。这个命令的效果是：在宿主机的`/var/lib/docker`目录下创建一个临时文件并把它链接到容器中的`/tmp`目录
- ADD ，拷贝文件并且重命名
- ENTRYPOINT ，为了缩短 Tomcat 的启动时间，添加`java.security.egd`的系统属性指向`/dev/urandom`作为 ENTRYPOINT

2、打包发布为远程docker镜像

```sh
./mvnw clean # 先清除一下
./mvnw package # 打包项目为jar包，顺利的话就生成了target文件夹和jar包
./mvnw install dockerfile:build # 可以详细的看到构建过程
docker images # 查看刚刚打包好的镜像 
```

> Maven是一个常用的构建工具，但是Maven的版本和插件的配合并不是那么完美，有时候你不得不切换到一个稍微旧一些的版本，以保证所有东西正常工作。Maven虽然没有官方的Wrapper，但是有一个第三方的Wrapper可以使用。mvnw允许你在没有Maven安装的情况下运行Maven项目。

3、运行镜像

**方式一：使用docker运行**

```sh
docker run -p 8080:9666 -t 398bigdata/config-server:latest
```

- -p：端口映射，8080表示主机端口，9666代表Docker容器运行端口
- -t：为容器重新分配一个伪输入终端

**方式二：使用docker-compose运行**

新建docker-compose.yml文件:
```yml
version: '2.0'
services:
  config-server:
    image: 398bigdata/config-server:0.0.1-SNAPSHOT
    network_mode: "host"
    environment:
      - port=9966
      - git=https://git.dev.tencent.com/weijunfang/shantoucredit.git
      - username=***
      - password=***
```

- version：Compose目前为止有三个版本分别为Version 1,Version 2,Version 3,Compose区分Version 1和Version 2（Compose 1.6.0+，Docker Engine 1.10.0+）。Version 2支持更多的指令。Version 1没有声明版本默认是"version 1"。Version 1将来会被弃用。
- config-server：服务名称
- image：从指定的镜像中启动容器，可以是存储仓库、标签以及镜像 ID
- network_mode：网络模式，用法类似于 Docker 客户端的 `--net` 选项，`host模式`容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口，不需要做端口映射。
- environment：添加环境变量，可以使用数组或字典。一般 `arg` 标签的变量仅用在构建过程中。而 `environment` 和 `Dockerfile` 中的 `ENV` 指令一样会把变量一直保存在镜像、容器中。

启动镜像：
```sh
docker-compose up -d
```

## 参考

[Spring Boot 2 (四)：使用 Docker 部署 Spring Boot](http://www.ityouknow.com/springboot/2018/03/19/spring-boot-docker.html)