---
title: "我家云L1 PRO"刷 Armbian 作为 V2ray 旁路网关
comments: true
date: 2021-01-02 16:42:19
categories:
tags:
  - v2ray
  - 旁路网关
  - ARM
  - docker
---
# "我家云L1 PRO"刷 Armbian 作为 V2ray 旁路网关

我一直以来都是使用"蜗牛星际"100m网口的主机作为旁路网关, 最近更换联通250M宽带
100M网口作为旁路口显然无法跑满带宽了. 恰好手里一个很久之前用玩客云和别人交换的
"我家云L1 PRO",查询参数后得知这个小主机居然是1000M网口. "我家云L1 PRO" 十分
省电,功耗只有5w.于是我果断上网搜索了类似的刷机方案,成功利用我家云实现了旁路网关.
下面是网上各种刷机资料教程的汇总.如果你想使用虚拟机制作旁路网关且系统debian,那么
本教程可供参考

这里方案我将使用 docker 来允许v2ray 客户端. docker 运行v2ray客户端的损耗几乎
可以忽略不记.因为我们运行 v2ray 容器时会使用主机网络栈.简单来说对于操作系统来讲就
是多了几个if else.

## 工具+材料
## 刷入 Armbian

## 安装配置旁路网关
### 安装docker

* 配置源  

信任 Docker 的 GPG 公钥:  
```shell script
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
ARM机器 添加docker-ce ARM 源:  
```shell script
$ echo "deb [arch=armhf] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian \
     $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list
```


X86 机器 添加docker-ce X86 源:  
```shell script
$ sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian \
   $(lsb_release -cs) \
   stable"
```
参考 [docker-ce tuna源](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)
    
* 安装并启动docker  
```shell script
$ sudo apt-get update
$ sudo apt-get install docker-ce
$ systemctl enable docker
$ systemctl start docker
```


### 启动 v2ray 
* 下载镜像  

v2ray 的服务端和客户端是共用一个文件/docker镜像. v2ray 换了维护者后,docker 镜像终于支持了ARM版的.
```shell script
docker pull v2fly/v2fly-core
```

* 配置文件准备

v2ray 的配置文件是最难理解,不是v2ray的重度用户很难理解配置文件.这里我直接给出模板,大家参考模板填写.
各位仅需修改 outbounds 数组中 tag=proxy 的对象即可.但是这样还是太困难了,大家买机场借本都是用二维
码或者base64的分享码导入手机APP使用.所以这里给大家介绍一个简单方式(电脑版客户端请自行测试): 
1. 二维码/分享码导入 v2rayNG
2. v2rayNG安卓app 导出完整配置
3. 拷贝json文件中 outbounds 数组中 tag=proxy 的对象
4. 覆盖如下模板 outbounds 数组中 tag=proxy 的对象 
5. 其他字段请勿修改.
```shell script
$ vim /root/v2ray/config.json
{
  "inbounds": [
    {
      "tag": "transparent",
      "port": 12345,
      "listen": "0.0.0.0",
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp,udp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      },
      "streamSettings": {
        "sockopt": {
          "tproxy": "tproxy"
        }
      }
    },
    {
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true,
        "userLevel": 8
      },
      "sniffing": {
        "destOverride": [
          "http",
          "tls"
        ],
        "enabled": true
      },
      "tag": "socks"
    },
    {
      "listen": "127.0.0.1",
      "port": 10809,
      "protocol": "http",
      "settings": {
        "userLevel": 8
      },
      "tag": "http"
    }
  ],
  "outbounds": [
    # 各位仅需要定制 tag=proxy 的字段即可
    {
      "tag": "proxy",
      "mux": {
        "concurrency": -1,
        "enabled": false
      },
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "xxxx.xxx",
            "port": 80,
            "users": [
              {
                "alterId": 2,
                "id": "xxxxxx",
                "level": 8,
                "security": "auto"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "",
        "wssettings": {
          "connectionReuse": true,
          "headers": {
            "Host": "xxxx.com"
          },
          "path": "/path"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "UseIP"
      },
      "streamSettings": {
        "sockopt": {
          "mark": 255
        }
      }
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "http"
        }
      }
    },
    {
      "tag": "dns-out",
      "protocol": "dns",
      "streamSettings": {
        "sockopt": {
          "mark": 255
        }
      }
    }
  ],
  "dns": {
    "servers": [
      "8.8.8.8",
      "1.1.1.1",
      "114.114.114.114",
      {
        "address": "223.5.5.5",
        "port": 53,
        "domains": [
          "geosite:cn",
          "ntp.org",
          "$myserver.address"
        ]
      }
    ]
  },
  "policy": {
    "levels": {
      "8": {
        "connIdle": 300,
        "downlinkOnly": 1,
        "handshake": 4,
        "uplinkOnly": 1
      }
    },
    "system": {
      "statsInboundUplink": true,
      "statsInboundDownlink": true
    }
  },
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [
      {
        "type": "field",
        "inboundTag": [
          "transparent"
        ],
        "port": 53,
        "network": "udp",
        "outboundTag": "dns-out"
      },
      {
        "type": "field",
        "inboundTag": [
          "transparent"
        ],
        "port": 123,
        "network": "udp",
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": [
          "223.5.5.5",
          "114.114.114.114"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": [
          "8.8.8.8",
          "1.1.1.1"
        ],
        "outboundTag": "proxy"
      },
      {
        "type": "field",
        "domain": [
          "geosite:category-ads-all"
        ],
        "outboundTag": "block"
      },
      {
        "type": "field",
        "protocol": [
          "bittorrent"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": [
          "geoip:private",
          "geoip:cn"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": [
          "geosite:cn"
        ],
        "outboundTag": "direct"
      }
    ]
  }
}
```

* 启动 v2ray 

```shell script
docker run -d --network=host --name v2ray -v /root/v2ray:/etc/v2ray  v2fly/v2fly-core   v2ray -config=/etc/v2ray/config.json
```

* 检查代理是否正常工作  

是否正常启动  
```shell script
$ docker logs -f v2ray
A unified platform for anti-censorship.
2021/01/01 14:22:38 [Info] v2ray.com/core/main/jsonem: Reading config: /etc/v2ray/config.json
2021/01/01 14:22:40 [Warning] v2ray.com/core: V2Ray 4.33.0 started
```
代理是否生效  
```shell script
$ export http_proxy=http://127.0.0.1:10809
$ export https_proxy=http://127.0.0.1:10809
$ curl www.google.com -I
  HTTP/1.1 200 OK
  Connection: close
  Transfer-Encoding: chunked
$ unset http_proxy=http://127.0.0.1:10809
$ unset https_proxy=http://127.0.0.1:10809
```

### 开启NAT转发,添加策略路由

```shell script
$ echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf && sysctl -p
$ ip rule add fwmark 1 table 100
$ ip route add local 0.0.0.0/0 dev lo table 100
```

### 配置 iptables 规则
这里的ip网段请参考自己LAN网段填写,否则旁路网关无法生效.我这里是`192.168.1.0/24`,
如果你家是其他的网段请注意修改 
```shell script
#友情增加环境变量
export LAN="192.168.1.0/24"
iptables -t mangle -N V2RAY
iptables -t mangle -A V2RAY -d 127.0.0.1/32 -j RETURN
iptables -t mangle -A V2RAY -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A V2RAY -d $LAN -p tcp -j RETURN
iptables -t mangle -A V2RAY -d $LAN -p udp ! --dport 53 -j RETURN 
iptables -t mangle -A V2RAY -p udp -j TPROXY --on-port 12345 --tproxy-mark 1 
iptables -t mangle -A V2RAY -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1 
iptables -t mangle -A PREROUTING -j V2RAY 
```

### ip rule规则,iptables规则开机自动重载
ip rules 规则,iptables 规则在设备重启后会消失.所以这里将
设置开机自启动脚本. 给 `/etc/rc.local` 文件赋予执行权限.
如果没有该文件请自行创建. 在文件中添加如下配置.请注意修改LAN
地址.
```shell script
$ chmod a+x /etc/rc.local
$ vim /etc/rc.local
###友情增加环境变量
export LAN="192.168.1.0/24"
docker restart v2ray
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100
iptables -t mangle -N V2RAY
iptables -t mangle -A V2RAY -d 127.0.0.1/32 -j RETURN
iptables -t mangle -A V2RAY -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A V2RAY -d $LAN -p tcp -j RETURN
iptables -t mangle -A V2RAY -d $LAN -p udp ! --dport 53 -j RETURN 
iptables -t mangle -A V2RAY -p udp -j TPROXY --on-port 12345 --tproxy-mark 1 
iptables -t mangle -A V2RAY -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1 
iptables -t mangle -A PREROUTING -j V2RAY 
###
```

### 配置静态 IP
为了避免重启后网关IP变动,我们需要配置静态IP.请按照自己的 LAN 网段
进行配置.我这里配置 `192.168.1.254`
```shell script
$ vim /etc/network/interfaces
# Network is managed by Network manager
auto lo
auto eth0
allow-hotplug eth0
iface eth0 inet static
address 192.168.1.254
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 192.168.1.1
```
重启主机生效
```shell script
$ reboot
```


## 客户端侧配置
有些家用无线路由支持配置: 下发网关,DNS 等配置.如果在无线路由器上配置下
发网关以及DNS,那么设备连接上无线路由器后会自动获取到你在无线路由器上配置
的网关以及DNS. openwrt 无线路由器可以在LAN口配置中增加 dhcp-option 
配置网关和DNS
```shell script
dhcp_option '3,192.168.1.254'
dhcp_option '6,114.114.114.114,8.8.8.8'
```
但并不是每个人都有这个条件,所以还是将说明一下客户端如何配置旁路网关地址
### windows

控制面板->网络和 Internet->网络连接  
* WLAN  
 
右键->属性->TCP/IPv4
配置如下图:  

* 以太网  
 
右键->属性->TCP/IPv4
配置如下图:  


### Android
* 设置->WLAN->已连接热点
配置如下:


## 结束
当你认真的完成这个教程的每一个步骤后,相信你已经能在手机,电脑等设备上无缝进行翻墙了.
我给出的 v2ray 模板配置文件已经配置了自动分流.由于TCP,UDP报文都要经过v2ray处理,
所以使用旁路网关玩游戏延迟会变高.客户端配置静态ip将网关还原默认的网关.

## 参考文档

[docker-ce tuna源](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)
[透明代理(TPROXY)](https://guide.v2fly.org/app/tproxy.html#%E8%AE%BE%E7%BD%AE%E7%BD%91%E5%85%B3)
[我家云刷OMV记录](https://zhuanlan.zhihu.com/p/101705429?utm_source=cn.wiz.note)