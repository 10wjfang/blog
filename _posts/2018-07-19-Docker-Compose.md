cloud-favorite
================

在线收藏夹是一个使用Spring boot开发的开源网站，用户可以随时随地收藏一个网站，可以分类收藏。

## Docker部署

### 安装docker环境

安装

```sh
yum install docker
```

安装完成后，启动docker服务，并设置开机启动

``` sh
systemctl start docker.service
systemctl enable docker.service
```

输入``docker -version``查看是否安装成功

### 安装JDK

```sh
yum -y install java-1.8.0-openjdk*
```

配置环境变量，打开``vi /etc/profile``，``:$``移动到最后一行，添加内容

```sh
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-8.b10.el7_5.x86_64
export PATH=$PATH:$JAVA_HOME/bin
```

注：JAVA_HOME的路径根据实际路径修改

修改完成后，使其生效

```sh
source /etc/profile
```

输入``java -version``查看是否安装成功

### 安装MAVEN

下载Maven

```sh
## 下载
wget http://mirrors.shu.edu.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz
## 解压
tar zxvf apache-maven-3.5.4-bin.tar.gz
## 移动
mv apache-maven-3.5.4 /usr/local/maven3
```

修改环境变量，在``/etc/profile``添加内容

```sh
export MAVEN_HOME=/usr/local/maven3
export PATH=$PATH:$MAVEN_HOME/bin
```

修改完成后，使其生效

```sh
source /etc/profile
```

输入``mvn -version``查看安装是否成功

### 安装Docker Compose

```sh
#下载
sudo curl -L https://github.com/docker/compose/releases/download/1.20.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
#安装
chmod +x /usr/local/bin/docker-compose
#查看版本
docker-compose version
```

### 下载源码

```sh
## 下载
wget https://github.com/10wjfang/cloud-favorite/archive/v0.1.zip
## 解压
unzip v0.1.zip
## 启动项目
cd cloud-favorite-0.1
docker-compose up -d
```