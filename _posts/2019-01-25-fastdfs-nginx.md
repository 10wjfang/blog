---
layout: post
title: FastDFS+Nginx+fastdfs-nginx-module服务器配置
date: 2019-1-25 17:15:24
catalog: true
tags:
    - FastDFS
    - Nginx
---

## 为什么要用Nginx？

FastDFS的客户端API来进行文件的上传、下载、删除等操作。同时通过FastDFS的HTTP服务器来提供HTTP服务。但是FastDFS的HTTP服务较为简单，无法提供负载均衡等高性能的服务，所以使用Nginx，fastdfs-nginx-module可以重定向连接到源服务器取文件,避免客户端由于复制延迟的问题，出现错误。

## 提前准备

安装FastDFS，见上节安装步骤。

## 安装Nginx

1、下载Nginx

下载地址：`http://nginx.org/en/download.html`，下载1.15以上版本比较好。

2、下载fastdfs-nginx-module

下载地址：`https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.20.tar.gz`

3、解压缩

```sh
apt-get install libssl-dev zlib1g-dev libpcre3-dev
tar -zxvf nginx-1.15.8.tar.gz
tar -zxvf V1.20.tar.gz
mv nginx-1.15.8 /opt
mv fastdfs-nginx-module-1.20 fastdfs-nginx-module
```

4、安装fastdfs-nginx-module

```sh
./configure --add-module=/opt/fastdfs-nginx-module/src
make
make install
```

## 修改配置文件

1、复制文件

```sh
cp mod_fastdfs.conf /etc/fdfs/ 
cp /opt/fastdfs-5.11/conf/http.conf /etc/fdfs/
cp /opt/fastdfs-5.11/conf/mime.types /etc/fdfs/
```

2、修改`mod_fastdfs.conf`配置文件：

```sh
vi /etc/fdfs/mod_fastdfs.conf 
base_path=/data/fastdfs/logs
#存放log的路径 
tracker_server=127.0.0.1:22122 
#指定tracker服务器及端口 
url_have_group_name = true 
#这个很重要，在URL中包含group名称
store_path0=/data/fastdfs/storage
#存储文件的路径 
storage_server_port=13580 
#与storage的配置端口保持一致 
#保存后退出 
```

3、创建文件夹

```sh
mkdir /data/fastdfs/logs
```

4、修改`nginx.conf`配置文件：

```
cd /usr/local/nginx/conf
vi nginx.conf
```

添加以下内容：

```
location /group1 {
    root   /data/fastdfs/storage/data;
    ngx_fastdfs_module;
}
```

5、重启Nginx

```sh
/usr/local/nginx/sbin/nginx -s stop
/usr/local/nginx/sbin/nginx
```

## FAQ

**问题1**：`/usr/local/include/fastdfs/fdfs_define.h:15:27: 致命错误：common_define.h：没有那个文件或目录`.

**解决：**

修改配置文件：
```sh
vi fastdfs-nginx-module-1.20/src/config
```
修改内容：
```
ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"
CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"
```

## 参考

[Ubuntu 14.04下部署FastDFS 5.08+Nginx 1.9.14](https://www.linuxidc.com/Linux/2016-07/133485.htm)

[FastDFS+nginx+fastdfs-nginx-module服务器配置](https://blog.csdn.net/jun2016425/article/details/53572088)

[FastDFS+Nginx(单点部署)事例](https://www.cnblogs.com/cnmenglang/p/6251696.html)