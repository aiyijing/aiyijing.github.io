---
title: ceph 15.2.4 安装
comments: true
date: 2020-08-15 16:58:24
categories:
  - 知识备忘
tags:
  - ceph
---

## 1 主机信息以及预备环境

### 1.1 主机信息

* centos 7.8 minimal
* 3 台主机 hostname 分别为: ceph1 ceph2 ceph3
* 3 台主机互相配置免密登录
* 3 台主机均添加 Host 信息

PS: Host 信息及其重要,如果未同步可能造成安装失败

```bash
# /etc/hosts
192.168.10.2  ceph1
192.168.10.3  ceph2
192.168.10.4  ceph3
```

* 磁盘信息

```bash
ceph1 /dev/vdb
ceph2 /dev/vdb
ceph3 /dev/vdb
```

### 1.2 预备环境

预备环境所有主机均需满足

#### 1.2.1 安装 python-setuptools

centos 7.8 minimal 未预装 python-setuptools
pecan, werkzeug 为ceph依赖项

```bash
yum install python-setuptools -y
pip3 install pecan werkzeug
```

#### 1.2.2 安装 ntp

```bash
yum install ntp ntpdate ntp-doc -y
systemctl start ntpd
systemctl status ntpd
```

#### 1.2.3 使用 epel-release

```bash
yum install -y yum-utils
yum install --nogpgcheck -y epel-release
```

#### 1.2.3 源设置

使用阿里源

```bash
sudo vim /etc/yum.repos.d/ceph.repo

[ceph]
name=ceph
baseurl=https://mirrors.aliyun.com/ceph/rpm-15.2.4/el7/x86_64/
gpgcheck=0
priority =1

[ceph-noarch]
name=cephnoarch
baseurl=https://mirrors.aliyun.com/ceph/rpm-15.2.4/el7/noarch/
gpgcheck=0
priority =1

[ceph-source]
name=cephsource
baseurl=https://mirrors.aliyun.com/ceph/rpm-15.2.4/el7/SRPMS/
gpgcheck=0
priority =1
```

```bash
yum makecache
```

## 2 ceph安装

ceph 的安装通过 ceph-deploy 完成, 所以所有的安装过程均可以在一台主机上完成
此处我们选择: ceph1

### 2.1 ceph 软件包安装

```bash
yum install -y ceph ceph-deploy ceph-radosgw
```

### 2.2 创建 ceph 集群

* 创建配置文件

```bash
# ceph 配置文件
cd /etc/ceph/
ceph-deploy new ceph1 ceph2 ceph3
```

* 子网配置

```bash
cat >> /etc/ceph/ceph.conf << EOF
public network = 192.168.10.0/24
osd pool default size = 3
EOF
```

* 部署 ceph

让 ceph2 ceph3 安装 ceph 相关包

```bash
# 设置环境变量,让安装过程使用阿里源
export CEPH_DEPLOY_REPO_URL=https://mirrors.aliyun.com/ceph/rpm-15.2.4/el7
export CEPH_DEPLOY_GPG_URL=http://mirrors.aliyun.com/ceph/keys/release.asc
ceph-deploy install ceph1 ceph2 ceph3
```

* 初始化 mon

```bash
ceph-deploy mon create-initial
```

* 同步 keyring

```bash
ceph-deploy admin ceph1 ceph2 ceph3
```

* 创建 manager daemon

```bash
ceph-deploy mgr create ceph1 ceph2 ceph3

```

* 创建 OSD

```bash
ceph-deploy disk zap ceph1   /dev/vdb
ceph-deploy disk zap ceph2   /dev/vdb
ceph-deploy disk zap ceph3   /dev/vdb

ceph-deploy osd create  ceph1 --data /dev/vdb
ceph-deploy osd create  ceph2 --data /dev/vdb
ceph-deploy osd create  ceph3 --data /dev/vdb

# dd if=/dev/zero of=/dev/vdb bs=512K count=1
# ceph-deploy osd create --data /dev/loop0 $HOSTNAME
```

* 创建MDS

```bash
ceph-deploy mds create ceph1 ceph2 ceph3
```

到处 ceph 集群已经可以正常使用了

### 2.3 ceph 文件存储测试

* 创建文件系统

```bash
ceph osd pool create cephfs_data 64
ceph osd pool create cephfs_metadata 64
ceph fs new myfs cephfs_metadata cephfs_data
```

* ceph1 挂载测试

```bash
mount -t ceph 192.168.10.2:6789:/ -o name=admin,secret= ${keyRing}
```

## 3 ceph 回滚安装,卸载

回滚安装: 指的是清除 ceph 集群配置文件.官网的说法是通过如下命令搞定,但是实际情况还是需
要手动清理所有节点的 /etc/ceph

```bash
ceph-deploy purgedata ceph1 ceph2 ceph3
ceph-deploy forgetkeys
```

卸载: 将删除 ceph 软件,集群配置信息

```bash
ceph-deploy purge ceph1 ceph2 ceph3
```

## 4 问题

* 重装遇到设备遇到 device busy
  在重装过程中,通过 ceph-deploy purge 卸载 ceph 并不会删除业务数据,业务数据依然
  保存在 /var/lib/ceph/osd 下. 这时候磁盘被挂载到 /var/lib/ceph/osd/ceph-{x}
  目录下.需要手动卸载磁盘并且格式化

```
umount /var/lib/ceph/osd/ceph-{x}
rm -fr /var/lib/ceph
dd if=/dev/zero of=/dev/vdb bs=512K count=1
```

* mds 进程重启
  mds 是一个十分占用内存的进程, 出故障后基本能通过重启恢复.

```bash
[root@ceph1 ~]# systemctl list-units|grep ceph-mds
ceph-mds@ceph1.service                                                      loaded active running   Ceph metadata server daemon
ceph-mds.target                                                             loaded active active    ceph target allowing to start/stop all ceph-mds@.service instances at once
[root@ceph1 ~]# systemctl restart ceph-mds@ceph1.service
```
