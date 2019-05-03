---
title: 进入暗网-Tor v3 搭建教程
comments: true
categories: Linux 服务搭建
tags:
  - tor
  - 暗网
abbrlink: 3602e88e
date: 2019-04-05 12:14:27
---

2019年1月 Tor 官方组织发布了 Tor v3 的稳定版本，同时 Brower Tor 的更新，许多的暗网组织都进行网络的升级。洋葱域名 .onion 也由16位扩展到 56位。目前v2网络还可以正常的访问。但是升级到 V3 是迟早的事情，但是国内许多的镜像源都每有进行 Tor 的更新。以下仅仅以 Ubuntu 16.04 TLS 为例，说明 Tor v3 的安装过程。
<a name="88210852"></a>
### 准备工作

1. 安装 apt-transport-https，让 APT 支持 https

```shell
apt update
apt install apt-transport-https
```

2. 添加APT源，编辑 `/etc/apt/sources.list` 将以下源写入文件末尾

```
deb https://deb.torproject.org/torproject.org xenial main
deb-src https://deb.torproject.org/torproject.org xenial main
```

3. 安装gpg key

```shell
curl https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | gpg --import
gpg --export A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89 | apt-key add -
```

<a name="e655a410"></a>
### 安装
先更新APT包管理器，然后进行tor安装（谨记）
```
apt update
apt install tor deb.torproject.org-keyring
```
<a name="d41d8cd9"></a>
##### 
<a name="7ed5a600"></a>
### 软件配置
通过以上的安装Tor 代理其实已经启动的，可以通过 `netstat -tuln | grep 9050`看到服务器已经启动。为了增加安全性，我们现在要对他进行个人设置。<br />生成自己的 hash密码<br />1.生成 hash-password
```
tor --hash-password password
```

2.修改配置文件 `/etc/tor/torrc`<br />   把注释中的ControlPort9051取消注释<br />   把注释中的HashedControlPassword取消注释，后面的密码改成由上一步生成的密码<br />3.重新启动Tor 代理

```
/etc/init.d/tor restart
```

<a name="433531fd"></a>
### 结语
通过以上的配置我们已经通过tor默认监听的9050端口使用sock5代理。下一篇博文将会介绍如何将sock5转换为浏览器可以用的HTTP代理。当然你也可以通过浏览器插件的方式使用sock5代理。
