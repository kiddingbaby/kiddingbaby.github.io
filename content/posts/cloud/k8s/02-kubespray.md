---
title: "K8s: 02-基于 Kubespray 部署高可用 Kubernetes 集群"
date: 2023-01-01T10:00:00+08:00
lastmod: 2025-06-05T12:00:00+08:00
description: "通过 Kubespray 快速构建支持多节点、高可用、负载均衡与外部 etcd 的 Kubernetes 集群，适用于生产环境部署。"
categories: ["云平台"]
tags: ["Kubernetes", "Kubespray", "集群部署", "高可用"]
series: ["K8s"]
keywords: ["Kubespray 高可用", "Kubernetes 生产部署", "多节点集群", "Kubernetes 集群搭建"]
ShowToc: true
TocOpen: true
draft: false
---

## 为什么要使用 Kubespray

使用 Kubespray 之前，我们首先要熟悉 [kubeadm 部署 Kubernetes 集群的基本流程]({{< relref "posts/cloud/k8s/01-kubeadm.md" >}})。

kubeadm 对环境准备、运维自动化支持有限，Kubespray 在其内部通过 Ansible 自动化编排 kubeadm 相关命令，并在此基础上支持多节点、高可用集群快速部署，同时兼顾集群生命周期管理，适合生产环境和复杂网络环境下的批量部署与维护。

## 如何构建高可用集群

首先需要了解构建高可用 Kubernetes 集群的整体流程：

📌注：此流程适用于私有云内网、离线环境、HA 架构下的 Kubernetes 集群搭建，公有云环境建议搭配云厂商的服务进行实践。

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
1. 搭建 etcd 外部集群（推荐使用官方二进制，设置 mTLS）
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
   * Ingress Controller（可替换）
   * Dashboard（可选）
1. 证书管理
1. 集群验证与健康检查
1. 日志与监控

从上面的列表我们可以发现，kubeadm 并没有涵盖整个高可用集群部署流程，例如缺乏对服务器初始化、外部 etcd 集群搭建、kube-apiserver 负载均衡配置等环节的支持。以上这些 Kubespray 都可以帮助我们来实现，利用 Ansible 脚本也可以大幅减少人工操作失误。

## 实践案例

下面我们将使用 Kubespray 在实验环境中来尝试构建高可用 Kubernetes 集群。

核心组件版本：

```yaml
Python: 3.11.2
Ansible: 2.16.14
Kubespray: 2.27.0
Kubernetes: 1.31.7
etcd (外部集群): 3.5.19 
```

Kubespray 依赖 Ansible，因此我们首先要在主控机上部署一个 Python 环境，并安装 Ansible 相关依赖。

1. 在服务器拉取稳定版本代码。（仓库当前采用主干开发模式，尚未提供稳定的 patch tag，建议使用 `release-<version>`）

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

1. 拷贝一份集群环境的配置，我这里会命名为 **sandbox**，企业中实际部署建议使用 `dev/test/staging/prod` 来规范命名

   ```bash
      cp -rfp inventory/sample inventory/sandbox
   ```

1. 修改 `inventory.ini` 清单文件，下面是一个高可用场景下的清单文件示例：

   ```bash
   etcd1 ansible_host=192.168.0.101 ip=192.168.0.101 etcd_member_name=etcd1
   etcd2 ansible_host=192.168.0.102 ip=192.168.0.102 etcd_member_name=etcd2
   etcd3 ansible_host=192.168.0.103 ip=192.168.0.103 etcd_member_name=etcd3

   kube-cp1 ansible_host=192.168.0.150 ip=192.168.0.150
   kube-cp2 ansible_host=192.168.0.151 ip=192.168.0.151
   kube-cp3 ansible_host=192.168.0.152 ip=192.168.0.152

   kube-node1 ansible_host=192.168.0.153 ip=192.168.0.153
   kube-node2 ansible_host=192.168.0.154 ip=192.168.0.154
   kube-node3 ansible_host=192.168.0.155 ip=192.168.0.155

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

1. 由于实验环境有限，我这里仅采用了 3 节点部署：

   ```bash
   [kube_control_plane]
   kube1 ansible_host=192.168.0.150 ip=192.168.0.150 etcd_member_name=etcd1
   kube2 ansible_host=192.168.0.151 ip=192.168.0.151 etcd_member_name=etcd2
   kube3 ansible_host=192.168.0.152 ip=192.168.0.152 etcd_member_name=etcd3

   [etcd:children]
   kube_control_plane

   [kube_node]
   kube1 ansible_host=192.168.0.150 ip=192.168.0.150
   kube2 ansible_host=192.168.0.151 ip=192.168.0.151
   kube3 ansible_host=192.168.0.152 ip=192.168.0.152  
   ```

1. 根据你的需求修改 `inventory/sandbox/group_vars/` 下的配置文件：

   ```bash
   ## `group_vars/all/`

   | 文件名                 | 说明                                                |
   | ---------------------- | --------------------------------------------------- |
   | `all.yml`              | 全局配置，控制运行时、证书、仓库等                  |
   | `containerd.yml`       | containerd 运行时配置                               |
   | `docker.yml`           | Docker 配置（不推荐）                               |
   | `cri-o.yml`            | CRI-O 运行时配置，适用于需要使用 CRI-O 的场景       |
   | `coreos.yml`           | 针对 CoreOS 系统的特殊配置，其他系统无需使用        |
   | `offline.yml`          | 离线部署相关配置，包括本地仓库地址、镜像预加载等    |
   | `etcd.yml`             | etcd 配置，包括内外部集群切换、证书参数、备份策略等 |
   | `<cloud_provider>.yml` | 视情况而定                                          |

   ## `group_vars/k8s_cluster/`
   
   | 文件名                   | 说明                                                                      |
   | ------------------------ | ------------------------------------------------------------------------- |
   | `k8s_cluster.yml`        | 核心集群参数，包括集群名称、Kubernetes 版本、控制面 VIP、LB 地址等        |
   | `addons.yml`             | 控制是否启用核心插件，如 Metrics Server、Ingress Controller、Dashboard 等 |
   | `k8s-net-<cni>.yml`      | CNI 插件高级配置。                                                        |
   | `kube_control_plane.yml` | 控制面节点相关参数配置，如证书 SAN、负载均衡设置等                        |
   ```

1. 接下来就可以一键部署集群了

   ```bash
   ansible-playbook -i inventory/sandbox/ -u $USERNAME -b -v --private-key=~/.ssh/id_rsa cluster.yml
   ```

1. 部署完成后，可以进行简单的测试

   ```bash
   kubectl top nodes
   kubectl get pods -A
   ```

离线部署可以参考这篇文章：[Kubespray 中的离线部署方案]({{< relref "posts/cloud/k8s/03-kubespray-offline.md" >}})。

## 其他建议

为了保持对官方稳定版本的持续追踪，建议将 Kubespray 作为子模块集成到企业的主代码库中，并采用 Git Submodule 管理 Kubespray 仓库。这样做既可以稳定获取官方的更新，又可以方便维护自定义扩展。

对于 Kubernetes 资源的自定义扩展，个人建议仅采用 Kustomize 或 Helm 进行管理，对于 Kubespray 本身尽可能不要做过多的二次开发。
