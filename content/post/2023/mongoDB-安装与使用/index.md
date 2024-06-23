---
title: mongoDB 安装与使用
comments: true
categories:
- Linux
tags:
- mongodb
date: 2019-03-01 15:24:15
---


mongoDB 的适用场景非常广泛。虽然不支持事务，但是自带的如findAndModify已经可以解决很多多线程，多进程难题。在支持的并发量上，虽然远不如SQL类数据库。所以也特别的依赖特定的业务场景，比如信息量十分巨大文档形式的数据。一下将展示如何在Linux下安装和启用外网访问mongoDB数据库
<a name="ab3615a5"></a>
### 下载安装包
安装包建议去官方网站下载，虽然如Centos ubunut等仓库源都有相关的快捷安装，但是我建议从官方网站下载 <br />[下载mongoDB](https://www.mongodb.com/download-center#community)

```shell
wget https://repo.mongodb.org/apt/ubuntu/dists/xenial/mongodb-org/4.0/multiverse/binary-amd64/mongodb-org-server_4.0.6_amd64.deb
```

根据系统，选择相关的软件安装（本文选择:version4.06  ubuntu 16.04  package 一定要选择server）<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/279029/1552224093040-6b4eb1e5-8f8b-4eac-9585-3f2f0eba27ea.png#align=left&display=inline&height=287&name=image.png&originHeight=506&originWidth=932&size=37167&status=done&width=529)

<a name="481dd3fe"></a>
### 安装mongoDB
打开终端：

```shell
sudo dpkg -i mongodb-org-server_4.0.6_amd64.deb
```

<a name="35dd6eac"></a>
### 配置mongoDB
mongoDB的配置文件在` /etc/mongod.conf`
```
vim /etc/mongod.conf
```
修改 mongo的绑定网卡和端口

```shell
# network interfaces
net:
port: 27017
bindIp: 0.0.0.0 #表示监所有地址.本地访问指定：127.0.0.1
```
重启mongoDB

```shell
/etc/init.d/mongodb restart
```
检查

```
netstat -tuln |grep 27017
```
可以看到程序已经监听0.0.0.0 地址

