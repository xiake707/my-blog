---
title: "kubelet、CRI 与容器运行时：Pod 如何真正启动"
date: 2026-09-14
lastmod: 2026-09-14
draft: false
description: "从 Pod 绑定到 Node 开始，梳理 kubelet、CRI、containerd、CNI 与 runc 之间的职责边界。"
categories:
  - Kubernetes
tags:
  - kubelet
  - containerd
  - CRI
  - CNI
featured: true
showToc: true
---

Scheduler 完成绑定，只代表 Kubernetes 已经决定 Pod 应该运行在哪个 Node，并不代表容器已经启动。真正把声明变成 Linux 进程的是目标节点上的 kubelet 与容器运行时。

## kubelet 的职责

kubelet 是每个 Node 上的节点代理。它持续观察分配给本节点的 Pod，并让本机实际状态尽量符合 API Server 中的期望状态。

它不会自己实现镜像管理、namespace、cgroup 和容器进程创建，而是通过 CRI 调用容器运行时：

```text
kubelet
   ↓ CRI
containerd / CRI-O
   ↓ OCI
runc
   ↓
Linux process
```

这种接口边界避免 Kubernetes 与某一种运行时实现绑定。

## Pod Sandbox 与共享网络

一个 Pod 可以包含多个容器。它们需要共享网络 namespace、Pod IP 和部分运行环境，因此运行时通常先创建 Pod Sandbox。

```text
Pod network namespace
        │
   ┌────┴────┐
   │         │
 nginx    sidecar
```

同一 Pod 中的容器可以通过 `localhost` 通信，但不能在同一个端口上重复监听。

## 启动过程

kubelet 处理一个新 Pod 时，典型流程可以概括为：

1. 创建 Pod Sandbox。
2. 通过 CNI 配置网络与 Pod IP。
3. 准备 Volume 和挂载点。
4. 根据拉取策略准备镜像。
5. 创建并启动业务容器。
6. 执行 startup、readiness 和 liveness 探针。
7. 把容器状态同步回 API Server。

这不是一次性任务。kubelet 会持续检查状态，并在容器退出或探针失败时根据 Pod 策略采取行动。

## 排障边界

理解组件边界后，可以更快缩小问题范围：

| 现象 | 优先检查 |
| --- | --- |
| Pod 一直 Pending，未绑定 Node | Scheduler、资源请求、亲和性、污点、Events |
| Pod 已绑定但停留在 ContainerCreating | kubelet、CNI、Volume、镜像拉取 |
| ImagePullBackOff | 镜像地址、凭据、网络、拉取策略 |
| CrashLoopBackOff | 应用退出原因、探针、配置、`logs --previous` |
| Node NotReady | kubelet、容器运行时、网络与节点资源 |

例如：

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
kubectl logs <pod-name> --previous
```

命令只是观察手段，关键是先判断问题发生在调度之前、节点准备阶段，还是应用进程启动之后。

## 总结

Scheduler 决定“去哪台机器”，kubelet 负责“让这台机器执行”，containerd 与 runc 负责“创建真实容器”。把这三层职责分开，是理解 Kubernetes 节点侧故障的基础。
