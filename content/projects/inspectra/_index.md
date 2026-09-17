---
title: "inspectra"
date: 2026-09-14
draft: false
description: "面向 Linux 主机的 Go 并发巡检工具，用于演示项目页面的信息结构。"
layout: "project"
featured: true
tech:
  - Go
  - JavaScript
  - Vue3
  - Vite
  - Element Plus
  - ECharts
  - Axios
  - Mysql/MariaDB
  - Docker
  - Nginx
  - SSH
weight: 10
---

本页负责介绍项目整体目标。架构、并发模型和使用方法等内容拆分在下方的项目文档中。

## 项目目标

将重复的主机检查流程抽象为可配置任务，通过受控并发执行远程命令，并统一汇总成功、失败和超时结果。

## 源码仓库

[github仓库](https://github.com/xiake707/inspectra)

## 核心能力

- 批量读取主机与检查项配置
- 控制并发数量和单任务超时
- 设计数据结构，将检查结果高效规律的存储到数据库中
- 设计后端 `api` 为前端接口提供条件
- 设计前端页面，让检查数据生动的呈现

## 架构思路

```text
前端接口
   ↓
后端应用
   ↓
 数据库

```

## 技术关注点

重点关注 `goroutine` 生命周期、超时取消、错误传播、连接复用和敏感信息处理，而不是简单地并发执行一组命令。
