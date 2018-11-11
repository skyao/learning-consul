---
date: 2018-11-10T10:30:00+08:00
title: 安装
weight: 200
description : "介绍Consul的安装"
---


## 下载安装

从官方下载对应操作系统的版本：

https://www.consul.io/downloads.html

consul不需要安装，简单解压缩下载文件即可。

## 简单启动

以最简单的方式启动一个 consul server 以便简单体验，或者开发使用。

```bash
cd consul
./consul agent -server -data-dir ./server1 -bind=127.0.0.1 -ui
```

- `-data-dir` 参数指定数据存储目录
- `-bind` 参数指定绑定的地址，有多个网络接口时需要指定
- `-ui` 参数指定开启UI界面，这样可以通过 `http://localhost:8500/ui` 这样的地址访问 consul 自带的web UI 界面。


