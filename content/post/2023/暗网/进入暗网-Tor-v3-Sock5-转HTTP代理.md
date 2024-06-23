---
title: 进入暗网-Tor v3 Sock5 转HTTP代理
comments: true
categories:
- Linux
tags:
- tor
date: 2019-04-04 15:16:03
---

在上一篇文章中我们介绍了 Tor v3 的搭建教程，Tor V3 默认只提供 TCP端口为 9050 Sock5代理。HTTP代理工作在应用层，仅仅为 HTTP 协议提供服务（ps：当然也可以通过 HTTP 协议代理转为 Sock5 代理） 。sock 代理分很多种，Sock 3.4.5等，Sock 代理工作在 OSI 七层网络模型的会话层。它可以为任何的工作在会话层之上的程序提供服务，包括浏览器、游戏、视频传输等等应用程序。从笔者的角度讲 Sock5 是绝对的优于HTTP代理，只是由于前期程序框架依赖 HTTP 代理故需要将 Sock5 转换为HTTP.
<a name="d52b3df2"></a>
#### 环境以及工具
系统：Ubuntu 16.04<br />工具：Python Python-pip
<a name="d41d8cd9"></a>
####
<a name="e6ecfb1f"></a>
#### 安装Python2.7以及python-pip

```shell
sudo apt update # 更新源
sudo apt install python
sudo apt install python-pip
```
<a name="a9484aef"></a>
#### 安装polipo

```shell
pip install polipo
```

<a name="4cc35e97"></a>
#### 配置polipo
polipo的配置文件默认在/etc/polipo/config

```shell
#服务器监听地址和端口配置
proxyAddress = "0.0.0.0"
proxyPort = 8118

# If you do that, you'll want to restrict the set of hosts allowed to
# connect:

# allowedClients = "127.0.0.1, 134.157.168.57"
# allowedClients = "127.0.0.1, 134.157.168.0/24"
# 客户端访问控制
allowedClients = 127.0.0.1, 0.0.0.0/0
allowedPorts = 1-65535

# Uncomment this if you want your Polipo to identify itself by
# something else than the host name:

proxyName = "localhost"

# Uncomment this if there's only one user using this instance of Polipo:

cacheIsShared = false


#Tor V3 Sock5代理地址
socksParentProxy = "localhost:9050"
#代理类型
socksProxyType = socks5
##########一下配置默认，和缓存以及权限控制有关
chunkHighMark = 67108864
diskCacheRoot = ""
localDocumentRoot = ""
disableLocalInterface = true
disableConfiguration = true
dnsUseGethostbyname = yes
disableVia = true
censoredHeaders = from,accept-language,x-pad,link
censorReferer = maybe
maxConnectionAge = 5m
maxConnectionRequests = 120
serverMaxSlots = 8
serverSlots = 2
tunnelAllowedPorts = 1-65535

```

1 配置polipo连接到sock5服务器

```shell
#Tor V3 Sock5代理地址
socksParentProxy = "localhost:9050"
#代理类型
socksProxyType = socks5
##########一下配置默认，和缓存以及权限控制有关
```

2 如果你想通过IPV6地址访问代理服务器请向这样设置

```
proxyAddress = "::0" IPV6 监听地址
proxyPort = 8118 #polipo http 代理端口
```

3 客户端的IP限制,` 0.0.0.0/0 `代表任意主机可以访问，请按照实际需求设置，多个地址段用","隔开

```
# 客户端访问控制
allowedClients = 127.0.0.1, 0.0.0.0/0
allowedPorts = 1-65535
```

<a name="e0ae5e26"></a>
#### 重启polipo

```
/etc/init.d/polipo restart
```


此后就可以通过HTTP代理愉快的使用Tor v3 的http代理了。如果无法成功使用，请检查访问配置以及防火墙配置
