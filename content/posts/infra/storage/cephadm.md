---
title: "基于 cephadm 部署高可用 Ceph 存储集群"
date: 2024-01-01T10:00:00+08:00
lastmod: 2025-06-07T10:00:00+08:00
description: "通过 cephadm 工具部署企业级高可用 Ceph 集群，支持多节点管理、自动化运维、Dashboard UI，适用于生产环境。"
categories: ["基础设施"]
tags: ["Ceph", "cephadm", "高可用", "分布式存储", "企业实践"]
series: ["部署方案"]
keywords: ["cephadm 高可用", "Ceph 存储集群", "Ceph 自动化部署"]
ShowToc: true
TocOpen: true
draft: true
---

## cephadm 是什么

Ceph 是一款企业级开源分布式存储系统，具备高可扩展性和强一致性。自 Octopus (15.2.x) 起，Ceph 引入 cephadm 作为官方推荐的部署与生命周期管理工具，基于容器实现自动化安装、升级、运维等操作。

相比传统的 ceph-deploy 和手动部署，cephadm 支持更强的自动化、服务编排、高可用性管理。另外一个可选方案是 ceph-ansible，但项目自从 2022 年 8 月起就已经不再发布新版本，社区也鼓励迁移到 cephadm，这里不再考虑。此外，在 Kubernetes 集群 Rook 提供了更云原生的部署方式来集成 Ceph。

那么，是使用更云原生的 Rook，还是使用官方主推的 cephadm 呢？

## 部署架构选型

在 Kubernetes 集群中集成 Ceph，Rook 是当前社区主流、最易上手的解决方案。然而，在现实场景中，Rook 也有一些明显的局限性：

* Ceph 对底层硬件性能和网络延迟十分敏感，而 Rook 又依赖于 Kubernetes 集群本身的稳定性，排障链路变长，极端场景下可能需要额外考虑 K8s 调度、网络、Pod 生命周期等多个维度，排障难度会大幅增加
* Rook 的典型模式是每个 K8s 集群部署一套独立的 Ceph 集群，虽然解耦性好，但是存储资源在集群间难以服用，资源成本和运维复杂度都不小
* Rook 只适合云原生场景，不兼容裸金属、私有云、公有云等混合架构下，无法作为统一的存储平台

在这里，个人更推荐使用官方主推的 cephadm 工具来部署 ceph 集群，通常来说，对于 Dev/Test/Staging 等非生产环境，可共用一套由 cephadm 部署的中小型 Ceph 集群，对于 Prod，则是单独一套由 cephadm 部署的 ceph 集群。具体做法是，使用 podman + systemd 管理外部 Ceph 集群，Rook 只作为 Kubernetes 存储消费端接入（即 external cluster）。这种方式可以很好的避免 Kubernetes 本身的不稳定影响，且降低了重复部署成本，也可以很好的适配混合架构。

## Ceph 如何作为统一存储平台

Ceph 支持块（RBD）、文件（CephFS）、对象（RGW）等多种存储协议：

| 协议类型        | 场景                                |
| --------------- | ----------------------------------- |
| RBD 块存储      | MySQL、PostgreSQL、MongoDB 等数据库 |
| CephFS 文件存储 | 共享挂载、日志存储、NFS 服务        |
| RGW 对象存储    | S3、备份、AI 模型管理               |

在常规情况下，Ceph 可以作为统一的底层存储平台，满足大多数容器化应用对块、文件和对象存储的需求。然而，对于某些 I/O 延迟极其敏感的场景（如 etcd、Redis、Kafka 等），可以考虑以下存储策略：

* Local PV + StatefulSet：使用节点本地硬盘（SSD/HDD）并配置节点亲和，适用于对性能敏感的服务，提供高 IOPS 和低延迟。
* K8s 外部部署/云托管：追求 K8s 集群无状态，将存储服务与 K8s 完全解耦，K8s 集群变动、迁移、升级时，存储服务无感知，提高可维护性和弹性。

## 实践案例

下面我们将使用 cephadm 在 HomeLab 中来尝试构建一个三节点的高可用 Ceph 集群，并在 Kubernetes 集群中通过 Rook External 模式接入。

核心组件版本：

```yaml
OS: Debian GNU/Linux 12 (bookworm)
Podman: 4.3.1
Ceph: 19.2.2 (squid)
Kubernetes: 1.31.7
Rook: v1.17.0
```

📌注：强烈推荐使用 Podman 代替 Docker，Podman 支持 rootless 模式、无守护进程（daemonless）运行，且可原生集成 systemd 单元服务，系统亲和性更强。由于 Red Hat 同时是 Ceph 与 Podman 的主要维护方，使用 Podman 能获得更好的兼容性、安全性与官方支持保障。

部署前确认以下事项：

* 三台裸机或虚拟机
* 每个节点配置至少为 Ceph 准备一块数据盘（无需格式化）
* 操作系统支持 Python3、Systemd、Podman/Docker、LVM2
* 已进行时间同步

具体部署流程：

1. ssh 到主控节点，直接安装 cephadm（也可以使用 curl 方式），不用担心发行版中 cephadm 版本过旧的问题，这里只是为了通过该命令配置仓库。

```bash
apt install -y cephadm
```

1. 通过 cephadm 添加 Ceph 官方最新稳定版仓库，并安装最新的 cephadm 依赖包

```bash
cephadm add-repo --release squid
cephadm install
```

1. 检查 PATH 变量

```bash
which cephadm
```

成功将返回以下信息：

```bash
/usr/sbin/cephadm
```

TODO

1. 引导创建新集群（cephadm bootstrap --mon-ip *<mon-ip>*）
1. 启用 ceph cli
1. 添加主机到集群中
1. 添加 OSD 存储
1. 启用内存自动调节选项
1. 使用 Ceph

Rook External 接入等流程。
