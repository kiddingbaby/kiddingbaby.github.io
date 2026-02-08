---
title: "K8s: 01-kubeadm 部署与升级 Kubernetes 集群的基本流程"
date: 2023-01-01T09:00:00+08:00
lastmod: 2025-06-05T12:00:00+08:00
categories: ["云平台"]
tags: ["Kubernetes", "kubeadm", "集群部署", "集群升级"]
series: ["K8s"]
keywords: ["kubeadm", "Kubernetes 集群部署", "Kubernetes 升级", "高可用集群"]
ShowToc: true
TocOpen: false
draft: false
---

## kubeadm 简介

kubeadm 是一种受官方支持且符合 CNCF 标准的部署工具，专注于 Kubernetes 集群本身的初始化和配置。

kubeadm 提供了 kubeadm init 和 kubeadm join 命令，通过执行必要的操作即可启动和运行最小可用集群。按照设计，它只关注启动引导，而非配置机器。除此之外，kubeadm 还支持 upgrade、reset 等基础操作。

## kubeadm 的部署流程

核心流程如下：

* `kubeadm init ...` 执行一系列预检查和准备工作，并安装控制平面节点
* 用户设置 KUBECONFIG
* `kubectl apply -f ...` 安装网络插件
* `kubeadm join ...` 加入工作节点

非常容易理解，具体操作网上资源很多，不做赘述。

下面是 kubeadm 命令的详细介绍（本文仅介绍 init、join、upgrade、reset 几个核心命令）。

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
| show-join-command  | 显示控制平面节点和工作节点的加入命令             |                                                      |

## kubeadm join

### join control-plane

| phases                | 用途                                             | 额外说明                                                                           |
| --------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------- |
| preflight             | 加入前预检                                       |                                                                                    |
| control-plane-prepare | 加入前的准备工作                                 | 包括从 kubeadm-certs Secret 下载共享证书，生成证书、kubeconfig 和静态 Pod 清单文件 |
| kubelet-start         | 写入 kubelet 配置文件，并启动或重启 kubelet 服务 |                                                                                    |
| control-plane-join    | 加入控制平面                                     | 包括添加新的本地 etcd 成员、将节点标记为控制平面                                   |
| wait-control-plane    | 等待控制平面节点启动                             |                                                                                    |

### join workflow

这里参考了官方文档，给出了较为流程化的表述：

1. 节点默认使用 bootstrap token 和 CA 密钥哈希（`--discovery-token-ca-cert-hash`）通过 API Server 的 cluster-info ConfigMap，
   自动发现并验证控制平面，并下载必要的集群元信息。也可通过文件或 URL 预先配置，跳过自动发现过程。
2. 获取集群信息后，kubelet 启动 TLS 引导流程，使用 bootstrap token 与 API Server 进行临时身份验证，并提交 CSR（证书签名请求）。
3. 控制平面中的 controller-manager 自动批准 CSR，kubelet 获取正式客户端证书。
4. kubelet 使用该证书与 API Server 建立正式连接，完成节点注册。

## kubeadm upgrade

### upgrade plan

检查可升级到哪些版本，并验证你当前的集群是否可升级。该命令只能在存在 kubeconfig 文件 admin.conf 的控制平面节点上运行。

### upgrade apply

将 Kubernetes 集群升级到指定版本。

| Phase 名称      | 功能描述                                     | 作用简述                                                          |
| --------------- | -------------------------------------------- | ----------------------------------------------------------------- |
| preflight       | 升级前预检                                   | 校验节点硬件、软件环境和集群状态，确保升级可行                    |
| control-plane   | 升级控制平面                                 | 依次升级 kube-apiserver、controller-manager、scheduler 等关键组件 |
| upload-config   | 将 kubeadm 和 kubelet 配置上传到 ConfigMap   |                                                                   |
| kubelet-config  | 升级此节点的 kubelet 配置                    | 从集群中的 kubelet-config ConfigMap 下载                          |
| bootstrap-token | 配置启动引导令牌和 cluster-info 的 RBAC 规则 |                                                                   |
| addon           | 升级核心插件                                 | 升级 CoreDNS、kube-proxy 等插件保证集群功能正常                   |
| post-upgrade    | 运行升级后的任务                             | 校验集群状态，清理临时文件及过时配置                              |

### upgrade diff

显示哪些差异将被应用于现有的静态 Pod 资源清单。

### upgrade node

升级集群中某个节点的命令。

| Phase 名称     | 说明                                                                           |
| -------------- | ------------------------------------------------------------------------------ |
| preflight      | 升级前环境检查，确保节点状态符合升级要求。                                     |
| kubelet-config | 应用并更新 kubelet 配置文件，准备重启 kubelet。                                |
| control-plane  | 仅针对控制平面节点，升级关键组件（apiserver、controller-manager、scheduler）。 |
| addon          | 升级集群关键插件（如 CoreDNS、kube-proxy）保证功能兼容。                       |
| post-upgrade   | 升级后检查节点状态，清理临时数据，验证升级成功。                               |

**集群升级操作流程：**

1. 备份关键数据和配置
    * 备份 etcd 数据
    * 备份控制平面节点的 `admin.conf` 和所有 kubeconfig 文件
1. 检查升级计划

     ```bash
     kubeadm upgrade plan
     ```

1. 升级第一个控制平面节点
    * 升级 kubeadm

      ```bash
      apt install -y kubeadm=<version>
      ```

    * 执行第一个控制平面升级：

        ```bash
        kubeadm upgrade apply <version>
        ```

    * 升级 kubelet 和 kubectl，重启 kubelet：

        ```bash
        apt install -y kubelet=<version> kubectl=<version>
        apt-mark hold kubelet kubectl
        systemctl daemon-reload
        systemctl restart kubelet
        ```

    * 验证节点状态和集群健康：

        ```bash
        kubectl get nodes
        kubectl get pods -n kube-system
        ```

1. 逐一升级剩余控制平面节点
    * 升级 kubeadm（同上）
    * 执行剩余控制平面升级

        ```bash
        kubeadm upgrade node
        ```

    * 升级 kubelet/kubectl，重启 kubelet（同上）
    * 验证节点状态和集群健康
1. 逐一升级所有工作节点
    * 升级 kubeadm（同上）
    * 执行工作节点升级：

        ```bash
        kubeadm upgrade node
        ```

    * 升级 kubelet 和 kubectl，重启 kubelet（同上）
    * 验证节点状态
1. 升级集群插件和附加组件
    * 升级 CoreDNS（通常由 kubeadm 自动完成）
    * 升级 kube-proxy（通常由 kubeadm 自动完成，视情况而定）

        ```bash
        kubectl -n kube-system edit ds kube-proxy
        # 修改 image 字段，例如：
        # image: registry.k8s.io/kube-proxy:v1.31.9
        kubectl rollout status ds kube-proxy -n kube-system
        ```

    * 升级 CNI（视插件类似而定）
1. 查看集群节点、Pod、etcd 各组件状态，并检查日志

    ```bash
    kubectl get nodes -o wide
    kubectl get pods -A
    ETCDCTL_API=3 etcdctl --endpoints=... endpoint health
    ```

注：建议先 draining 待升级节点，逐节点滚动升级，确保集群始终保持高可用。

## kubeadm reset

尽力恢复通过 `kubeadm init` 或 `kubeadm join` 对此主机所做的更改。

| phases             | 用途               |
| ------------------ | ------------------ |
| preflight          | 重置预检           |
| remove-etcd-member | 移除本地 etcd 成员 |
| cleanup-node       | 清理节点           |

## 系统证书详情

K8s 系统证书默认存放在 `/etc/kubernetes/pki` 目录下，通常由 kubeadm 初始化和维护。

* 证书生成流程

    初始化前，可以通过 `--dry-run` 参数查看生成详情。

    ```bash
    kubeadm init phase certs all --dry-run
    ```

    示例输出：

    ```bash
    [certs] Using certificateDir folder "/etc/kubernetes/tmp/kubeadm-init-dryrun2727823051"
    [certs] Using existing ca certificate authority
    [certs] Using existing apiserver certificate and key on disk
    [certs] Using existing apiserver-kubelet-client certificate and key on disk
    [certs] Using existing front-proxy-ca certificate authority
    [certs] Using existing front-proxy-client certificate and key on disk
    [certs] Generating "etcd/ca" certificate and key
    [certs] Generating "etcd/server" certificate and key
    [certs] etcd/server serving cert is signed for DNS names [kube1 localhost] and IPs [192.168.0.150 127.0.0.1 ::1]
    [certs] Generating "etcd/peer" certificate and key
    [certs] etcd/peer serving cert is signed for DNS names [kube1 localhost] and IPs [192.168.0.150 127.0.0.1 ::1]
    [certs] Generating "etcd/healthcheck-client" certificate and key
    [certs] Generating "apiserver-etcd-client" certificate and key
    [certs] Generating "sa" key and public key
    ```

* 证书到期时间

    ```bash
    kubeadm certs check-expiration
    ```

    示例输出：

    ```bash
    CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
    admin.conf                 Jun 15, 2026 16:55 UTC   361d            ca                      no      
    apiserver                  Jun 15, 2026 16:55 UTC   361d            ca                      no      
    apiserver-kubelet-client   Jun 15, 2026 16:55 UTC   361d            ca                      no      
    controller-manager.conf    Jun 15, 2026 16:55 UTC   361d            ca                      no      
    front-proxy-client         Jun 15, 2026 16:55 UTC   361d            front-proxy-ca          no      
    scheduler.conf             Jun 15, 2026 16:55 UTC   361d            ca                      no      
    super-admin.conf           Jun 15, 2026 16:55 UTC   361d            ca                      no      

    CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
    ca                      Jun 13, 2035 16:55 UTC   9y              no      
    front-proxy-ca          Jun 13, 2035 16:55 UTC   9y              no      
    ```

## 非系统证书

上面提到的都是系统证书，除此之外的绝大部分非系统组件证书，都可以通过 Cert-manager 来管理，证书均为动态签发与续期，无需人工干预。

* 查看所有证书状态：

  ```bash
  kubectl get certificate -A
  ```

* 查看签发历史：

  ```bash
  kubectl get certificaterequest -AA
  ```
