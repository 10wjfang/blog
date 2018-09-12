---
layout: post
title: 手把手搭建Shadowsocks
date: 2018年9月12日10:02:12
catalog: true
tags:
    - Shadowsocks
---

## 搭建自己的Shadowsocks

### 购买VPS

帮瓦工 地址：[https://bwh1.net/index.php](https://bwh1.net/index.php)

VPS终身优惠6% : `BWH1ZBPVK`

现在购买的VPS没有一键安装Shadowsocks服务，所以需要自己手动安装。

机房选择：`优先选择香港机房，其次选择美国西海岸机房(CN2/直连方案)，稳定性有保障。`
美国西海岸4个[首选]：洛杉矶QNET、洛杉矶MCOM、硅谷费利蒙(Fremont)、凤凰城(Phoenix)

### 安装Shadowsocks

```
yum install wget net-tools -y && wget --no-check-certificate -O shadowsocks.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
chmod +x shadowsocks.sh
./shadowsocks.sh 2>&1 | tee shadowsocks.log
```
即可正常安装。 
卸载:`./shadowsocks.sh uninstall`

centos需要额外运行以下命令。

```
yum install wget net-tools -y
```

要不会出现 -bash: wget: command not found及I can not find the server pubilc Ethernet! 这两个错误。

设置密码、端口号、加密方式后即可。

常用命令:

```
启动：service shadowsocks start
停止：service shadowsocks stop
重启：service shadowsocks restart
状态：service shadowsocks status  
```

注意:防火墙打开用到的ss端口,否则是连不上ss的。 
编辑/etc/firewalld/zones下的public.xml，在里面添加:
```
<port protocol="udp" port="4400-4600"/>
<port protocol="tcp" port="4400-4600"/>
```

### 本机上配置shadowsocks

- 下载（各种版本的[Clients](https://shadowsocks.org/en/download/clients.html)）
首先去下一个shadowsocks for windows，[here(shadowsocks4.0.2)](https://github.com/shadowsocks/shadowsocks-windows/releases/download/4.0.2/Shadowsocks-4.0.2.zip)

- 在状态栏右击shadowsocks，勾选开机启动和启动系统代理，在系统代理模式中选择PAC模式，服务器->编辑服务器，用配置好的相应的ip、密码、加密方法填好，保存即可。

查看配置信息：
```sh
cat /etc/shadowsocks-libev/config.json
```
```
{
    "server":"0.0.0.0",
    "server_port":16987,
    "password":"123456",
    "timeout":300,
    "user":"nobody",
    "method":"aes-256-gcm",
    "fast_open":false,
    "nameserver":"8.8.8.8",
    "mode":"tcp_and_udp"
}
```