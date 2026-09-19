---
title: "Debug for error of pod with 'ImagePullBackOff'"
date: 2026-09-16
lastmod: 2026-09-16
draft: false
description: "使用kubelet describe 定位错误详情，一步一步验证，定位问题原因"
categories:
  - Kubernetes
tags:
  - kubelet
  - containerd
  - CRI
  - CNI
  - Error
featured: true
showToc: true
---
首先执行下面的命令确定当前的状态

```shell
xwang@xwangl:~/ComputerStudy/SRE/k8s$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-585c5579f6   0         0         0       3d
nginx-deployment-69858744c4   3         3         3       8m31s
nginx-deployment-7965c789bf   0         0         0       2d12h
nginx-deployment-9b798668d    0         0         0       2d12h
xwang@xwangl:~/ComputerStudy/SRE/k8s$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           3d
```
deployment管理的pod都是ready的状态说明pod状态正常。

再执行
```shell
xwang@xwangl:~/ComputerStudy/SRE/k8s$ kubectl get pods -l app=nginx
NAME                                READY   STATUS             RESTARTS      AGE
nginx-deployment-69858744c4-7t4c9   1/1     Running            0             9m25s
nginx-deployment-69858744c4-979wf   1/1     Running            0             9m25s
nginx-deployment-69858744c4-jvp47   1/1     Running            0             9m26s
nginx-pod                           0/1     ImagePullBackOff   2 (52m ago)   3d1h
```
可以看到名字为nginx-pod的pod目前的状态是`ImagePullBackOff` 从字面意思来看大致的意思是镜像拉取失败。于是我就使用以下命令精确报错原因

```shell
xwang@xwangl:~/ComputerStudy/SRE/k8s$ kubectl describe pods nginx-pod # 我只看event字段的信息。
...
...
Events:
  Type     Reason          Age                   From     Message
  ----     ------          ----                  ----     -------
  Normal   SandboxChanged  56m                   kubelet  Pod sandbox changed, it will be killed and re-created.
  Normal   Pulling         47m (x5 over 56m)     kubelet  spec.containers{nginx}: Pulling image "nginx:latest"
  Warning  Failed          47m (x5 over 55m)     kubelet  spec.containers{nginx}: Error: ErrImagePull
  Warning  Failed          9m58s (x12 over 55m)  kubelet  spec.containers{nginx}: Failed to pull image "nginx:latest": rpc error: code = DeadlineExceeded desc = failed to pull and unpack image "docker.io/library/nginx:latest": failed to resolve reference "docker.io/library/nginx:latest": failed to do request: Head "https://registry-1.docker.io/v2/library/nginx/manifests/latest": proxyconnect tcp: dial tcp 10.208.12.91:7890: i/o timeout
  Normal   BackOff         85s (x193 over 55m)   kubelet  spec.containers{nginx}: Back-off pulling image "nginx:latest"
  Warning  Failed          31s (x197 over 55m)   kubelet  spec.containers{nginx}: Error: ImagePullBackOff
```

发现报错信息中非常关键的信息
```shell
tcp: dial tcp 10.208.12.91:7890: i/o timeout
```
与`10.208.12.91:7890`通讯的时候超时了

因为我的系统做了Clash verge的系统代理在宿主机上`127.0.0.1:7890`，但这里的`Control plane`和`woker node`都是在容器中运行，所以我在容器的Containerd.service中添加了系统代理指向宿主机ip和端口。来实现系统代理

宿主机7890端口监听情况：
```shell
xwang@xwangl:~/ComputerStudy/SRE/k8s$ ss -lntup | grep 7890
udp   UNCONN 0      0                                              *:7890             *:*    users:(("verge-mihomo",pid=7552,fd=10))
tcp   LISTEN 0      4096                                           *:7890             *:*    users:(("verge-mihomo",pid=7552,fd=9))
```

宿主机ip

```shell
xwang@xwangl:~/ComputerStudy/SRE/k8s$ ifconfig
wlp0s20f3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.74.191  netmask 255.255.240.0  broadcast 192.168.79.255
        inet6 fe80::1abe:7b81:8975:8f0a  prefixlen 64  scopeid 0x20<link>
        ether 64:49:7d:65:c8:04  txqueuelen 1000  (Ethernet)
        RX packets 592745  bytes 748515992 (713.8 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 411182  bytes 124711774 (118.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

问题就可以找到了宿主机ip改变导致的node节点没有办法在通过之前的系统代理访问docker的官方仓库。

我的解决方案：

1. 重新创建集群，并在创建集群时生命下面的shell环境变量
```shell
export HTTP_PROXY=http://192.168.74.191:7890
export HTTPS_PROXY=http://192.168.74.191:7890
export http_proxy=http://192.168.74.191:7890
export https_proxy=http://192.168.74.191:7890
```
让后执行下面的命令
```shell
kind create --name k8s-study --config cluster-config.yaml
```
这样重新创建的集群，就会把刚刚生命的环境变量一同加载到容器节点中。

2. 使用`docker exec -it k8s-study-worker /bin/bash` 进入容器中修改containerd.service文件的环境变量为现在的宿主机ip代理。


这里我们要保留k8s的集群状态，所以我们走方案二：

>进入容器中查看当前服务的代理环境变量

执行命令`cat /proc/$(pidof containerd)/environ | tr '\0' '\n' | grep -i proxy`

```shell
xwang@xwangl:~/ComputerStudy/SRE/k8s$ docker exec -it k8s-study-worker /bin/bash
root@k8s-study-worker:/etc/systemd/system/containerd.service.d# cat /proc/$(pidof containerd)/environ | tr '\0' '\n' | grep -i proxy
HTTPS_PROXY=http://10.208.12.91:7890
HTTP_PROXY=http://10.208.12.91:7890
NO_PROXY=fc00:f853:ccd:e793::/64,172.20.0.0/16,localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,::1,10.96.0.0/16,10.244.0.0/16,k8s-study-control-plane,k8s-study-worker,.svc,.svc.cluster,.svc.cluster.local
no_proxy=fc00:f853:ccd:e793::/64,172.20.0.0/16,localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,::1,10.96.0.0/16,10.244.0.0/16,k8s-study-control-plane,k8s-study-worker,.svc,.svc.cluster,.svc.cluster.local
```

> 修改containerd的代理环境变量为目前的宿主机的真实ip

```shell
root@k8s-study-worker:/etc/systemd/system/containerd.service.d# pwd
/etc/systemd/system/containerd.service.d
root@k8s-study-worker:/etc/systemd/system/containerd.service.d# cat proxy.conf
[Service]
Environment="HTTP_PROXY=http://10.208.12.91:7890"
Environment="HTTPS_PROXY=http://10.208.12.91:7890"
Environment="no_proxy=fc00:f853:ccd:e793::/64,172.20.0.0/16,localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,::1,10.96.0.0/16,10.244.0.0/16,k8s-study-control-plane,k8s-study-worker,.svc,.svc.cluster,.svc.cluster.local"
```

执行以下命令进行修改
```shell
cat > proxy.conf << eof
[Service]
Environment="HTTP_PROXY=http://192.168.74.191:7890"
Environment="HTTPS_PROXY=http://192.168.74.191:7890"
Environment="http_proxy=http://192.168.74.191:7890"
Environment="https_proxy=http://192.168.74.191:7890"
Environment="no_proxy=fc00:f853:ccd:e793::/64,172.20.0.0/16,localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,::1,10.96.0.0/16,10.244.0.0/16,k8s-study-control-plane,k8s-study-worker,.svc,.svc.cluster,.svc.cluster.local"
eof
systemctl daemon-reload
systemctl restart containerd
```

>验证修改
```shell
cat /proc/$(pidof containerd)/environ
```
执行命令
```shell
root@k8s-study-worker:/etc/systemd/system/containerd.service.d# cat /proc/$(pidof containerd)/environ
HTTPS_PROXY=http://192.168.74.191:7890HTTP_PROXY=http://192.168.74.191:7890LANG=C.UTF-8NO_PROXY=fc00:f853:ccd:e793::/64,172.20.0.0/16,localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,::1,10.96.0.0/16,10.244.0.0/16,k8s-study-control-plane,k8s-study-worker,.svc,.svc.cluster,.svc.cluster.localPATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/binNOTIFY_SOCKET=/run/systemd/notifyUSER=rootINVOCATION_ID=ec3f050a98974301901c5eeb66214bceJOURNAL_STREAM=9:1709485SYSTEMD_EXEC_PID=3239MEMORY_PRESSURE_WATCH=/sys/fs/cgroup/system.slice/containerd.service/memory.pressureMEMORY_PRESSURE_WRITE=c29tZSAyMDAwMDAgMjAwMDAwMAA=http_proxy=http://192.168.74.191:7890https_proxy=http://192.168.74.191:7890no_proxy=fc00:f853:ccd:e793::/64,172.20.0.0/16,localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,::1,10.96.0.0/16,10.244.0.0/16,k8s-study-control-plane,k8s-study-worker,.svc,.svc.cluster,.svc.cluster.local
```
已经修改成功

回到宿主机上使用kubectl 调用api server 查看当前pod的状况
```shell
xwang@xwangl:~/ComputerStudy/SRE/k8s$ kubectl get pods -l app=nginx
NAME                                READY   STATUS    RESTARTS      AGE
nginx-deployment-69858744c4-7t4c9   1/1     Running   0             48m
nginx-deployment-69858744c4-979wf   1/1     Running   0             48m
nginx-deployment-69858744c4-jvp47   1/1     Running   0             48m
nginx-pod                           1/1     Running   3 (92m ago)   3d2h
```
全部运行了起来。
