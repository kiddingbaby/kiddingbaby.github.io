---
title: "基于 Kubespary 部署高可用 Kubernetes 集群"
date: ""
description: ""
categories: ["云平台"]
tags: ["Kubernetes", "Kubespray"]
series: ["部署方案"]
ShowToc: true
TocOpen: false
---

## 为什么要使用 Kubespray?

kubeadm 是一种受官方支持且符合 CNCF 标准的部署工具，专注于 Kubernetes 集群本身的初始化和配置，但是对环境准备、运维自动化支持有限。

Kuberspray 在此基础上集成了 Ansible 自动化编排，支持多节点、高可用集群快速部署，同时兼顾集群生命周期管理，适合生产环境和复杂网络环境下的批量部署与维护。

## 如何构建高可用集群？

在使用 Kubespary 之前，我们首先要熟悉 [kubeadm 部署 Kubernetes 集群的基本流程]({{< relref "posts/cloud/kubernetes/kubeadm.md" >}})，如果你看过 Kubespray 的实现，会发现其内部也是调用了 kubeadm 命令来完成部署的。

这里先列出构建高可用 Kubernetes 集群的整体流程：

<!-- 1. 集群准备：
   * 服务器准备
   * 网络连通测试
   * sudo 权限配置
   * ssh 配置
   * 安装 kubelet/kubeadm
   * 安装容器运行时
   * 离线镜像仓库配置（可选）
   * 安装 kubectl
2. 外部 etcd 集群搭建（systemd 管理）
3. 容器运行时安装
4. 负载均衡器
5. etcd 集群搭建
6. 控制平面节点初始化
7. 证书管理
8. 工作节点加入置
9. 网络插件部署
10. CoreDNS 部署
11. 集群验证与健康检查 -->

从上面的列表我们可以发现，kubeadm 并没有参与到整个高可用集群部署方案中，比如集群服务器上的必要准备工作，外部 etcd 集群的搭建，kube-apiserver 的负载均衡器配置等，而这些 Kubespray 都可以帮助我们来实现，利用 Ansible 脚本也可以大幅减少人工操作失误。

下面我们使用 Kubespray 来尝试构建高可用 Kubernetes 集群。

## 实践案例

核心组件版本：

```yaml
Kubespray: 2.27.0
Kubernetes: 1.31.7
etcd: 3.5.19 (独立集群)
```

