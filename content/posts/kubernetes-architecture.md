---
title: "从一次 Deployment 创建理解 Kubernetes 核心架构"
date: 2026-09-15
lastmod: 2026-09-15
draft: false
description: "沿着 kubectl、API Server、etcd、Scheduler、Controller 与 kubelet 的协作链路，理解 Kubernetes 如何把声明转化为运行中的 Pod。"
categories:
  - Kubernetes
tags:
  - kube-apiserver
  - etcd
  - Scheduler
  - kubelet
featured: true
showToc: true
---

理解 Kubernetes 的关键，不是孤立地记住组件名称，而是弄清楚一个声明从提交到落地，究竟经过了哪些控制环节。

假设我们提交一个包含三个副本的 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

接下来沿着这次变更，观察控制平面与工作节点如何协作。

## 集群的两个部分

Kubernetes 集群可以先抽象为控制平面和工作节点：

```text
                    Kubernetes Cluster

        ┌─────────────────────────────┐
        │        Control Plane        │
        │                             │
        │  API Server      etcd       │
        │  Scheduler       Controller │
        └──────────────┬──────────────┘
                       │
                 控制与状态同步
                       │
        ┌──────────────┴──────────────┐
        │                             │
┌───────▼────────┐           ┌────────▼───────┐
│ Worker Node 1  │           │ Worker Node 2  │
│ kubelet        │           │ kubelet        │
│ containerd     │           │ containerd     │
│ kube-proxy     │           │ kube-proxy     │
└────────────────┘           └────────────────┘
```

控制平面负责保存和协调期望状态，工作节点负责让容器真正运行。两者并不是一次性下发关系，而是持续观察和校正。

## API Server：统一入口

执行下面的命令时，`kubectl` 不会直接访问每台 Node，也不会直接修改 etcd：

```bash
kubectl apply -f deployment.yaml
```

请求首先进入 `kube-apiserver`。API Server 负责：

1. 验证请求格式与 API 版本。
2. 完成身份认证和权限检查。
3. 执行准入控制。
4. 持久化合法的对象状态。
5. 向其他控制组件提供观察集群状态的统一接口。

这使 API Server 成为集群内部的协调中心。Scheduler、Controller 和 kubelet 都通过它读取或更新状态，而不是绕过它随意操作存储。

## etcd：保存集群状态

etcd 是 Kubernetes 使用的分布式键值存储。这里保存的重点不是“当前有几个 Linux 进程”，而是 Kubernetes API 对象及其状态。

对这次 Deployment 而言，集群需要持久化的信息包括：

- 对象名称和命名空间
- 期望副本数
- Pod 模板
- Selector
- 当前观察到的状态

可以把关系简化为：

```text
kubectl / Controller / Scheduler / kubelet
                    ↓
               API Server
                    ↓
                   etcd
```

生产环境中不应该把 etcd 当作普通业务数据库直接使用，也不应该让业务组件绕过 API Server 修改集群状态。

## Controller：让实际状态追上期望状态

Deployment 创建后，并不是 API Server 亲自启动三个 Pod。Deployment Controller 会观察相关对象，发现期望状态与实际状态存在差距：

```text
期望副本：3
实际副本：0
差距：3
```

控制器随后创建 ReplicaSet，ReplicaSet Controller 再创建对应的 Pod 对象。这个过程体现了 Kubernetes 最核心的控制模式：

```text
观察当前状态
      ↓
比较期望状态
      ↓
执行一次校正
      ↓
继续观察
```

这个循环通常称为 reconcile。它解释了为什么删除一个由 Deployment 管理的 Pod 后，新的 Pod 会再次出现：控制器发现实际副本数减少，于是重新校正。

## Scheduler：为 Pod 选择 Node

新创建的 Pod 一开始还没有 `spec.nodeName`。Scheduler 只处理尚未绑定到 Node 的 Pod，大致经过三个阶段：

### 过滤

排除不满足条件的 Node，例如：

- 资源请求无法满足
- Node Selector 或亲和性不匹配
- 污点无法被容忍
- 存储或端口条件冲突

需要注意，Scheduler 主要依据 Pod 的 `requests` 判断资源，而不是简单查看某一时刻的 CPU 使用率。

### 打分

对过滤后剩余的 Node 进行评分，选择整体更合适的目标。

### 绑定

把选择结果写回 API Server：

```text
Pod web-xxx → Node worker-2
```

Scheduler 的职责到绑定为止，它不负责登录 Node 启动容器。

## kubelet：把 Pod 落到真实节点

每台工作节点都运行 kubelet。它持续观察分配给本节点的 Pod。当 kubelet 发现新的 Pod 后，会调用容器运行时接口准备实际运行环境：

```text
kubelet
   ↓ CRI
containerd
   ↓
runc
   ↓
Linux namespaces / cgroups / process
```

典型过程包括：

1. 创建 Pod Sandbox。
2. 调用 CNI 准备网络。
3. 挂载所需 Volume。
4. 拉取镜像。
5. 创建并启动业务容器。
6. 执行探针和状态检查。
7. 通过 API Server 汇报状态。

Kubernetes 负责的是编排和状态协调，真正创建 Linux 容器的仍然是底层容器运行时。

## 一次完整链路

把前面的步骤串起来：

```text
kubectl apply
      ↓
API Server 校验并保存 Deployment
      ↓
Deployment Controller 创建 ReplicaSet
      ↓
ReplicaSet Controller 创建 Pod
      ↓
Scheduler 为 Pod 选择 Node
      ↓
目标 Node 上的 kubelet 发现 Pod
      ↓
containerd / runc 创建容器
      ↓
kubelet 持续汇报状态
```

这条链路也是排障时的重要地图。如果 Pod 长时间 Pending，应优先观察调度条件和 Events；如果已经绑定 Node 但容器无法启动，则应继续检查 kubelet、镜像、存储、网络和容器运行时。

## 总结

可以用下面几句话概括核心职责：

| 组件 | 核心职责 |
| --- | --- |
| API Server | 集群 API 与统一协调入口 |
| etcd | 持久化 Kubernetes 对象状态 |
| Controller | 持续校正实际状态与期望状态 |
| Scheduler | 为未绑定的 Pod 选择 Node |
| kubelet | 让本节点的 Pod 真正运行并持续汇报状态 |

真正掌握这些关系后，再学习 Deployment、Service、探针和故障排查时，就不再只是记忆命令，而是能够判断问题发生在哪个控制环节。
