---
title: CentOS7下kubernetes单节点部署实验环境
comments: true
categories:
  - 知识备忘
tags:
  - kubernetes
abbrlink: d59be515
date: 2019-05-05 14:23:10
---

## Kubernetes

分布式容器集群管理系统, 主要针对各个抽象化资源进行管理,如基础资源: Volumes Pod Replication Service.
主要的软件包：
下载地址：[kubernetes release](https://github.com/kubernetes/kubernetes/releases)

|  主要文件说明                  |                         |
| ---------------------------- | ----------------------- |
| etcd                         |       分布式存储数据库     |
| kube-apiserver               |        认证 授权 准入     |
| kube-controller-manager      |        资源管理器         |
| kube-scheduler               |        资源调度器         |
| kubelet                      |         Pod构建          |
| kube-proxy                   |         负载均衡          |

其中Master节点需要安装也上所有软件,工作Node上需至少安装:kubelet kube-proxy.为了更加方便的学习kubernetes,所以本次教程仅仅使用单节点部署环境。当需要研究kube-scheduler kube-proxy的集群调度相关的功能时再进行Node的加入

## 软件安装
在CentOS7环境下:
```shell
# 更新源
yum update
# 安装软件,自动安装 kube-apiserver kube-controller-manager kube-scheduler kube-proxy
yum install -y etcd kubernetes  
# 自动安装依赖：docker-ce
```

## 系统环境准备

为了更加简单的部署,直接将防火墙关闭

* 防火墙策略 官网文档[document](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/)
* 禁用防火墙

```shell  
systemctl stop firewalld
```

* 禁止开机启动

```shell
systemctl disable firewalld
```

## apiserver 配置

若不需要建立API TOKEN 关闭KUBE_ADMISSION_CONTROL

```shell
# vim /etc/kubernetes/apiserver
# 禁用 KUBE_ADMISSION_CONTROL
#KUBE_ADMISSION_CONTROL
```

## 服务启动

顺序启动服务

```shell
systemctl start etcd
systemctl start docker
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy
```

## 墙内额外配置

* 墙内无法访问 hub.docker.io  
配置 docker 仓库镜像源  

```shell
# vim /etc/docker/daemon.json
{
"registry-mirrors": ["https://pee6w651.mirror.aliyuncs.com"]
}
```

* CentOS 7 默认 pause 镜像无法安装

```shell
# vim /etc/kubernetes/kubelet
# 更换为阿里云pause镜像
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.cn-beijing.aliyuncs.com/zhoujun/pause-amd64:3.1"
```

未完待续......
