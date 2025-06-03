---
title: "kubeadm 部署 Kubernetes 集群的基本流程"
date: ""
description: ""
categories: ["云平台"]
tags: ["Kubernetes", "kubeadm"]
series: ["部署方案"]
ShowToc: true
TocOpen: false
---

## kubeadm 简介

kubeadm 是一个提供了 kubeadm init 和 kubeadm join 的工具，通过执行必要的操作来启动和运行最小可用集群。按照设计，它只关注启动引导，而非配置机器。除此之外，kubeadm 还支持 upgrade、reset 等基础操作。

## kubeadm 的部署流程

核心流程如下：

* `kubeadm init ...` 执行一系列预检查和准备工作，并安装控制平面节点
* 用户设置 KUBECONFIG
* `kubectl apply -f ...` 安装网络插件
* `kubeadm join ...` 加入工作节点

非常好理解，下面是 kubeadm 命令的详细介绍（本文仅介绍 init、join、upgrade、reset 几个核心命令）。

## kubeadm init

| phases             | 用途                                             | 额外说明                                             |
| ------------------ | ------------------------------------------------ | ---------------------------------------------------- |
| preflight          | 预检                                             |                                                      |
| certs              | 生成证书                                         | 包括 CA、apiserver、etcd                             |
| kubeconfig         | 生成 kubeconfig 文件                             | 包括 admin、super-admin、kubelet、controller-manager |
| etcd               | 生成静态 Pod 清单文件（仅本地 etcd）             |                                                      |
| control-plane      | 生成静态 Pod 清单文件                            | 包括 apiserver、controller-manager、scheduler        |
| kubelet-start      | 写入 kubelet 配置文件，并启动或重启 kubelet 服务 |                                                      |
| upload-config      | 将 kubeadm 和 kubelet 的配置上传到 ConfigMap     |                                                      |
| upload-certs       | 证书上传到 kubeadm-certs                         |                                                      |
| mark-control-plane | 将节点标记为控制面                               | 添加标签和污点                                       |
| bootstrap-token    | 生成用于将节点加入集群的引导令牌                 |                                                      |
| kubelet-finalize   | TLS 引导成功后，更新与 kubelet 相关设置          | 启用 kubelet 客户端证书自动轮换（可选）              |
| addon              | 安装用于通过一致性测试所需的插件                 | CoreDNS 和 kube-proxy                                |
| show-join-command  | 显示控制平面和工作节点的加入命令                 |                                                      |

## kubeadm join

### join control-plane

| phases                | 用途                                             | 额外说明                                                                           |
| --------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------- |
| preflight             | 加入前预检                                       |                                                                                    |
| control-plane-prepare | 加入前的准备工作                                 | 包括从 kubeadm-certs Secret 下载共享证书，生成证书、kubeconfig 和静态 Pod 清单文件 |
| kubelet-start         | 写入 kubelet 配置文件，并启动或重启 kubelet 服务 |                                                                                    |
| control-plane-join    | 加入控制平面                                     | 包括添加新的本地 etcd 成员、将节点标记为控制平面                                   |
| wait-control-plane    | 等待控制平面启动                                 |                                                                                    |

### join workflow

这里参考了官方文档，给出了较为流程化的表述：

1. 节点默认使用 bootstrap token 和 CA 密钥哈希（`--discovery-token-ca-cert-hash`）通过 API Server 的 cluster-info ConfigMap，自动发现并验证控制平面，并下载必要的集群元信息。也可通过文件或 URL 预先配置，跳过自动发现过程。
2. 获取集群信息后，kubelet 启动 TLS 引导流程，使用 bootstrap token 与 API Server 进行临时身份验证，并提交 CSR（证书签名请求）。
3. 控制平面中的 controller-manager 自动批准 CSR，kubelet 获取正式客户端证书。
4. kubelet 使用该证书与 API Server 建立正式连接，完成节点注册。

## kubeadm upgrade

### upgrade plan

检查可升级到哪些版本，并验证你当前的集群是否可升级。该命令只能在存在 kubeconfig 文件 admin.conf 的控制平面节点上运行。

### upgrade apply

将 Kubernetes 集群升级到指定版本。

### upgrade diff

显示哪些差异将被应用于现有的静态 Pod 资源清单。

### upgrade node

升级集群中某个节点的命令。

## kubeadm reset

尽力恢复通过 `kubeadm init` 或 `kubeadm join` 对此主机所做的更改。

| phases             | 用途               |
| ------------------ | ------------------ |
| preflight          | 重置预检           |
| remove-etcd-member | 移除本地 etcd 成员 |
| cleanup-node       | 清理节点           |
