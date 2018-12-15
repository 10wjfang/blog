---
layout: post
title: 搭建 docker registry 私有仓库
date: 2018-12-14 17:24:17
catalog: true
tags:
    - Docker
---

## 安装Docker Registry

1、从docker hub拉取官方`registry`镜像。

```sh
docker pull registry
```

由于在国内，访问国外的网站速度可能会较慢，所以，最好为ubuntu添加加速器。修改`/etc/docker/daemon.json`文件：

```json
{
    "registry-mirrors": ["http://registry.docker-cn.com"]
}
```

## 启动Registryr容器

```sh
docker run -d   --name=my-docker-registry  --restart=always -p 5000:5000   -v  /opt/data/registry:/var/lib/registry/docker/registry/v2/repositories    registry

#说明：启动一个名字为 my-docker-registry-2 的容器，端口映射到宿主机的5000，挂载宿主机目录 /opt/data/registry 到容器的/var/lib/registry/docker/registry/v2/repositories ，用于存储 push 进去的镜像文件。
```

## 测试

1、从docker hub获取ubuntu 16.04镜像

```sh
docker pull ubuntu:16.04
```

2、将镜像重新打上tag

```sh
docker tag ubuntu:16.04 192.168.241.135:5000/ubuntu:16.04
```

3、将新打标签的镜像push到本地仓库

```sh
docker push 192.168.241.135:5000/ubuntu:16.04
```

> *可能会报错*：The push refers to repository [192.168.241.135:5000/ubuntu：16.04]
Get https://192.168.241.135:5000/v2/: http: server gave HTTP response to HTTPS client
> *解决办法*：在”/etc/docker/“目录下，创建”daemon.json“文件。在文件中写入：
> { "insecure-registries":["192.168.241.135:5000"] }

4、查看镜像是否已上传成功，访问：`http://192.168.241.135:5000/v2/_catalog`

返回：
```json
{
    "repositories": [
        "ubuntu"
    ]
}
```

5、从本地仓库拉取镜像

```sh
docker pull 192.168.241.135:5000/ubuntu:16.04
```

> 说明：docker 命令通过`.:`识别是本地仓库

## 参考

[在 ubuntu 搭建 docker registry 私有仓库](http://blog.51cto.com/hellocjq/2070884)

[docker registry push错误“server gave HTTP response to HTTPS client”](https://www.cnblogs.com/hobinly/p/6110624.html)