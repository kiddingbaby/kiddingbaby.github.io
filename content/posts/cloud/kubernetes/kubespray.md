---
title: "基于 Kubespray 部署高可用 Kubernetes 集群"
date: 2023-01-01T10:00:00+08:00
lastmod: 2025-06-05T12:00:00+08:00
description: "通过 Kubespray 快速构建 Kubernetes 高可用集群，支持多节点部署、负载均衡、外部 etcd，适用于生产环境。"
categories: ["云平台"]
tags: ["Kubernetes", "Kubespray", "集群部署", "高可用"]
series: ["部署方案"]
keywords: ["Kubespray 高可用", "Kubernetes 生产部署", "多节点集群", "Kubernetes 集群搭建"]
ShowToc: true
TocOpen: true
draft: false
---

## 为什么要使用 Kubespray?

kubeadm 是一种受官方支持且符合 CNCF 标准的部署工具，专注于 Kubernetes 集群本身的初始化和配置，但是对环境准备、运维自动化支持有限。

Kuberspray 在此基础上集成了 Ansible 自动化编排，支持多节点、高可用集群快速部署，同时兼顾集群生命周期管理，适合生产环境和复杂网络环境下的批量部署与维护。

## 如何构建高可用集群？

在使用 Kubespary 之前，我们首先要熟悉 [kubeadm 部署 Kubernetes 集群的基本流程]({{< relref "posts/cloud/kubernetes/kubeadm.md" >}})，因为其内部也是调用了 kubeadm 命令来完成部署的。

这里先列出构建高可用 Kubernetes 集群的整体流程：
**注：此流程适用于私有云内网、离线环境、HA 架构下的 Kubernetes 集群搭建，公有云环境建议搭配云厂商的服务进行实践。**

1. 集群准备：
   * 服务器准备
   * sudo 权限配置
   * ssh 配置
   * 防火墙与内核设置
   * 网络连通性验证
   * 时间同步
   * 主机名映射
   * 安装容器运行时
   * 安装 kubelet、kubeadm、kubectl
   * 离线镜像配置（可选）
1. 搭建 etcd 高可用集群（推荐使用官方二进制，外部部署）
1. 控制平面负载均衡
   * LB 自身的高可用性与故障转移策略
   * 为控制平面配置 VIP
1. 初始化首个控制平面节点
1. 加入其余控制平面节点
1. 加入工作节点
1. 部署 CNI 插件，配置网络策略
1. 安装核心插件
   * CoreDNS
   * kube-proxy（Cilium 可替换）
   * Metrics Server
   * Ingress Controller
   * Dashboard（可选）
1. 证书管理
1. 集群验证与健康检查
1. 日志与监控
   * Prometheus Stack
   * EFK Stack

从上面的列表我们可以发现，kubeadm 并没有参与到整个高可用集群部署方案中，比如集群服务器上的必要准备工作，外部 etcd 集群的搭建，kube-apiserver 的负载均衡器配置等。

以上这些 Kubespray 都可以帮助我们来实现，利用 Ansible 脚本也可以大幅减少人工操作失误。

下面我们使用 Kubespray 来尝试构建高可用 Kubernetes 集群，本文仅给出基本的部署流程。

## 实践案例

核心组件版本：

```yaml
Python: 3.11.2
Ansible: 2.16.14
Kubespray: 2.27.0
Kubernetes: 1.31.7
etcd（外部集群）: 3.5.19 
```

Kubespary 依赖 Ansible，因此我们首先要在主控机上部署一个 Python 环境，并安装 Ansible 相关依赖。

1. 在服务器拉取稳定版本代码。（建议从 release 分支拉，官方目前采用主干开发模式，暂无 patch tag）

```bash
   VENVDIR=/opt/kubespray-venv
   KUBESPRAYDIR=/opt/kubespray

   python3 -m venv $VENVDIR
   source $VENVDIR/bin/activate

   git clone https://github.com/kubernetes-sigs/kubespray.git
   cd kubespray
   git switch release-2.27
```

1. 安装 Ansible 及相关依赖：

```bash
   cd $KUBESPRAYDIR
   pip install -U -r requirements.txt
```

1. 拷贝一份集群环境的配置，我这里会命名为 **sanzbox**，企业中实际部署建议使用 `dev/test/staging/prod` 来规范命名

```bash
   cp -rfp inventory/sample inventory/sandbox
```

1. 修改 `inventory.ini` 清单文件，下面是一个高可用场景下的清单文件示例：

```bash
etcd1      ansible_host=192.168.10.11 etcd_member_name=etcd1
etcd2      ansible_host=192.168.10.12 etcd_member_name=etcd2
etcd3      ansible_host=192.168.10.13 etcd_member_name=etcd3
kube-cp1   ansible_host=192.168.10.21
kube-cp2   ansible_host=192.168.10.22
kube-cp3   ansible_host=192.168.10.23
kube-node1 ansible_host=192.168.10.31
kube-node2 ansible_host=192.168.10.32
kube-node3 ansible_host=192.168.10.33

[etcd]
etcd1
etcd2
etcd3

[kube_control_plane]
kube-cp1
kube-cp2
kube-cp3

[kube_node]
kube-node1
kube-node2
kube-node3
```

1. 根据你的需求修改 `inventory/sandbox/group_vars/` 下的配置文件：

```bash
## `group_vars/all/`
| 文件名            | 说明                                                                       |
| ----------------- | -------------------------------------------------------------------------- |
| `all.yml`         | 全局配置，控制运行时类型、证书路径、仓库地址、时间服务器等，影响所有节点。 |
| `containerd.yml`  | containerd 运行时相关配置，如版本、镜像地址、安装方式等。                  |
| `docker.yml`      | Docker 运行时配置（不推荐使用，仅保留兼容性）。                            |
| `cri-o.yml`       | CRI-O 运行时配置，适用于需要使用 CRI-O 的场景。                            |
| `coreos.yml`      | 针对 CoreOS 系统的特殊配置，其他系统无需使用。                             |
| `offline.yml`     | 离线部署相关配置，包括本地仓库地址、镜像预加载等。                         |
| `etcd.yml`        | etcd 配置，包括内外部集群切换、证书参数、备份策略等。                      |
| `<云厂商相关>.yml | 视情况而定                                                                 |

## `group_vars/k8s_cluster/`
| 文件名                   | 说明                                                                        |
| ------------------------ | --------------------------------------------------------------------------- |
| `k8s_cluster.yml`        | 核心集群参数，包括集群名称、Kubernetes 版本、控制面 VIP、LB 地址等。        |
| `addons.yml`             | 控制是否启用核心插件，如 Metrics Server、Ingress Controller、Dashboard 等。 |
| `k8s-cluster.yml`        | 网络插件选择与基础配置，如 kube-proxy 模式、网络插件类型。                  |
| `k8s-net-<cni>.yml`      | CNI 插件高级配置。                                                          |
| `kube_control_plane.yml` | 控制面节点相关参数配置，如证书 SAN、负载均衡设置等。                        |
```

1. 接下来就可以一键部署集群了

```bash
ansible-playbook -i inventory/sandbox/ -u $USERNAME -b -v --private-key=~/.ssh/id_rsa cluster.yml
```

1. 部署完成后，可以进行简单的测试

```bash
kubectl top nodes
```
