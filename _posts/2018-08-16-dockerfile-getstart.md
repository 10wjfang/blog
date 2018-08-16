---
layout: post
title: Dockerfile介绍
date: 2018-8-16 11:31:26
catalog: true
tags:
    - Docker
---

## 简介

Dockerfile是由一系列命令和参数构成的脚本，这些命令应用于基础镜像并最终创建一个新的镜像。它们简化了从头到尾的流程并极大的简化了部署工作。Dockerfile从FROM命令开始，紧接着跟随者各种方法，命令和参数。其产出为一个新的可以用于创建容器的镜像。

## 语法

Dockerfile语法由两部分构成，注释和命令+参数

1. #Line blocks used for commenting
2. command argument argument ..

一个简单的例子：
```Dockerfile
# Print "Hello docker!"
RUN echo "Hello docker!"
```

## 基本命令

#### ADD

ADD命令有两个参数，源和目标。它的基本作用是从源系统的文件系统上复制文件到目标容器的文件系统。如果源是一个URL，那该URL的内容将被下载并复制到容器中。

```Dockerfile
# Usage: ADD [source directory or URL] [destination directory]
ADD /my_app_folder /my_app_folder 
```

#### RUN

RUN命令是Dockerfile执行命令的核心部分。它接受命令作为参数并用于创建镜像。不像CMD命令，RUN命令用于创建镜像（在之前commit的层之上形成新的层）。

```Dockerfile
# Usage: RUN [command]
RUN aptitude install -y riak
```

#### CMD

和RUN命令相似，CMD可以用于执行特定的命令。和RUN不同的是，这些命令不是在镜像构建的过程中执行的，而是在用镜像构建容器后被调用。

```Dockerfile
# Usage 1: CMD application "argument", "argument", ..
CMD "echo" "Hello docker!"
```

#### ENTRYPOINT

配置容器启动后执行的命令，并且不可被 docker run 提供的参数覆盖。将docker run 指令后面跟的内容当做参数作为ENTRYPOINT指令指定的运行命令的参数

每个 Dockerfile 中只能有一个 ENTRYPOINT，当指定多个时，只有最后一个起效。

ENTRYPOINT 帮助你配置一个容器使之可执行化，如果你结合CMD命令和ENTRYPOINT命令，你可以从CMD命令中移除“application”而仅仅保留参数，参数将传递给ENTRYPOINT命令。

```Dockerfile
# Usage: ENTRYPOINT application "argument", "argument", ..
# Remember: arguments are optional. They can be provided by CMD
# or during the creation of a container.
ENTRYPOINT echo
# Usage example with CMD:
# Arguments set with CMD can be overridden during *run*
CMD "Hello docker!"
ENTRYPOINT echo
```

#### ENV

ENV命令用于设置环境变量。这些变量以”key=value”的形式存在，并可以在容器内被脚本或者程序调用。这个机制给在容器中运行应用带来了极大的便利。

```Dockerfile
# Usage: ENV key value
ENV SERVER_WORKS 4
```

#### FROM

FROM命令可能是最重要的Dockerfile命令。该命令定义了使用哪个基础镜像启动构建流程。基础镜像可以为任意镜像。如果基础镜像没有被发现，Docker将试图从Docker image index来查找该镜像。FROM命令必须是Dockerfile的首个命令。

```Dockerfile
# Usage: FROM [image name]
FROM ubuntu 
```

#### EXPOSE

EXPOSE用来指定端口，使容器内的应用可以通过端口和外界交互。

```Dockerfile
# Usage: EXPOSE [port]
EXPOSE 8080
```

#### MAINTAINER

建议这个命令放在Dockerfile的起始部分，虽然理论上它可以放置于Dockerfile的任意位置。这个命令用于声明作者，并应该放在FROM的后面。

```Dockerfile
# Usage: MAINTAINER [name]
MAINTAINER authors_name 
```

#### USER

USER命令用于设置运行容器的UID。

```Dockerfile
# Usage: USER [UID]
USER 751
```

#### VOLUME

VOLUME命令用于让你的容器访问宿主机上的目录。

```Dockerfile
# Usage: VOLUME ["/dir_1", "/dir_2" ..]
VOLUME ["/my_files"]
```

#### WORKDIR

WORKDIR命令用于设置CMD指明的命令的运行目录。

```Dockerfile
# Usage: WORKDIR /path
WORKDIR ~/
```

## 构建镜像

进入Dockerfile所在目录，运行命令：
```sh
sudo docker build -t 镜像名称 .
```

注意：最后有个点用来表示当前目录

## 构建微服务Dockerfile例子

```Dockerfile
FROM java:8-jre
MAINTAINER Alexander Lukyanchikov <sqshq@sqshq.com>

ADD ./target/statistics-service.jar /app/
CMD ["java", "-Xmx200m", "-jar", "/app/statistics-service.jar"]

EXPOSE 7000
```