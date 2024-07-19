---
title: 使用 skopeo 同步镜像
comments: true
categories:
  - docker
tags:
  - 容器镜像
date: 2021-07-01 15:24:15
---

# 为什么使用 skopeo?
1. **独立的镜像管理工具**：不依赖于 Docker 或 Containerd，提供了更灵活的镜像管理方式。
2. **无痕同步**：在多仓库间同步镜像时不留痕迹，不会产生垃圾数据。
3. **广泛兼容性**：支持多种镜像格式，包括 OCI 和 Docker-archive 格式，兼容性非常强。
# 安装
### Ubuntu/Debian
```bash
apt-get -y install skopeo
```
### Ubuntu/Debian
```bash
yum -y install skopeo
```
# 使用场景
### 1. 两个镜像仓库间同步数据
```bash
skopeo copy --dest-tls-verify=false --src-tls-verify=false \
--override-arch amd64 --override-os linux \
docker://docker.io/nginx:1.27.0-perl \
docker://ghcr.io/aiyijing/nginx:1.27.0-perl
```
**参数说明**:  
--dest-tls-verify: 目的仓库非 HTTPS 或自签证书时需设置为 false  
--src-tls-verify: 源仓库非 HTTPS 或自签证书时需设置为 false  
--override-arch: Apple Silicon 架构下同步 Linux/AMD 镜像需将 arch 设置为 arm64，AMD64 架构下同步到 ARM 镜像同理设置为 arm64  
--override-os: 同上  
### 2. 将 docker-archive 包同步到镜像仓库
```bash
# docker save nginx:1.27.0-perl > nginx.1.27.0-perl.tar.gz
skopeo copy --dest-tls-verify=false \
docker-archive:nginx.1.27.0-perl.tar.gz \
docker://ghcr.io/aiyijing/nginx:1.27.0-perl
```
无需 docker load，不产生垃圾数据。
