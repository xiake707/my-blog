---
title: "kubectl logs 排障指南：从单容器到多副本"
date: 2026-09-13
lastmod: 2026-09-13
draft: false
description: "整理 kubectl logs 在单 Pod、多容器、多副本和 CrashLoopBackOff 场景中的常用方法与排障边界。"
categories:
  - SRE
tags:
  - Kubernetes
  - Logging
  - Troubleshooting
featured: true
showToc: true
---

`kubectl logs` 主要读取容器进程的标准输出和标准错误。它不会自动扫描容器内的所有日志文件，因此云原生应用通常应优先把日志输出到 stdout/stderr。

## 查看单个 Pod

```bash
kubectl logs web-7d8c9f4d5b-abcde
```

生产日志量较大时，应限制时间和行数：

```bash
kubectl logs web-7d8c9f4d5b-abcde --since=10m --tail=100
```

持续观察新日志：

```bash
kubectl logs -f web-7d8c9f4d5b-abcde
```

## 多容器 Pod

Pod 中存在多个容器时，通过 `-c` 指定容器：

```bash
kubectl logs web-7d8c9f4d5b-abcde -c nginx
kubectl logs web-7d8c9f4d5b-abcde -c log-agent
```

如果需要检查所有容器：

```bash
kubectl logs web-7d8c9f4d5b-abcde --all-containers=true
```

## 多副本应用

通过标签选择一组 Pod，并为每行增加来源前缀：

```bash
kubectl logs \
  -l app=web \
  --all-containers=true \
  --prefix=true \
  --since=5m \
  --tail=100
```

`--prefix=true` 很重要，否则即使看到了错误，也难以确认它来自哪个 Pod 和容器。

## 查看崩溃前日志

容器发生重启后，当前日志可能已经属于新一轮进程。使用 `--previous` 查看上一次运行实例：

```bash
kubectl logs web-7d8c9f4d5b-abcde --previous
```

这对 `CrashLoopBackOff` 场景尤其有用。

## logs 与 describe 如何选择

| 场景 | 优先手段 | 原因 |
| --- | --- | --- |
| Pending | `describe` 与 Events | 容器可能尚未创建 |
| ImagePullBackOff | `describe` 与 Events | 问题发生在镜像准备阶段 |
| Running 但返回 500 | `logs` | 应用进程已经运行 |
| CrashLoopBackOff | `describe` + `logs --previous` | 同时需要状态原因和崩溃输出 |

推荐先确认 Pod 所处阶段，再选择观察手段，而不是看到任何问题都直接执行 `kubectl logs`。
