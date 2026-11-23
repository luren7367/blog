---
title: "Docker 常用命令备忘"
slug: "docker-notes"
date: 2026-11-08
description: "平时最常用的一批 Docker 命令，按场景分类整理。"
image: "cover.png"
tags:
    - Docker
    - 运维
categories:
    - 技术
---

## 容器

```bash
docker ps -a                  # 查看所有容器
docker logs -f <name>         # 跟踪日志
docker exec -it <name> bash   # 进入容器
docker rm -f <name>           # 强制删除
```

## 镜像

```bash
docker images                 # 本地镜像
docker pull nginx:alpine      # 拉取
docker build -t app:1.0 .     # 构建
docker image prune -f         # 清理悬空镜像
```

## 磁盘占用

容器跑久了，磁盘很容易被日志和悬空镜像吃满：

```bash
docker system df              # 查看占用
docker system prune -a        # 清理（慎用，会删掉未使用的镜像）
```

## 小技巧

`docker compose` 里把 `restart: unless-stopped` 加上，机器重启后服务会自动拉起。
