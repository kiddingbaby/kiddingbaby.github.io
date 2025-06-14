---
title: "Ceph: 01-基于 cephadm 部署高可用 Ceph 存储集群"
date: 2025-06-12T10:00:00+08:00
lastmod: 2025-06-12T10:00:00+08:00
categories: ["基础设施"]
tags: ["Ceph", "cephadm", "高可用", "分布式存储", "企业实践"]
series: ["Ceph"]
keywords: ["cephadm 高可用", "Ceph 存储集群", "Ceph 自动化部署"]
ShowToc: true
TocOpen: true
draft: true
---

## 各种 Ceph 部署工具比较

Ceph 是一款企业级开源分布式存储系统，具备高可扩展性与强一致性。早期 Ceph 部署主要采用 ceph-deploy 配合 Ansible 的方式，目前，ceph-deploy 已逐步被淘汰，而 ceph-ansible 项目自 2022 年 8 月起停止发布新版本，社区建议迁移至 cephadm。从 Octopus（15.2.x）版本起，Ceph 官方引入并推荐使用基于容器的部署与运维工具 cephadm，用于统一管理集群的部署、升级与服务编排。

在 Kubernetes 环境中，Rook 提供原生方式来消费 Ceph 存储，是当前最主流的集成方案。Rook 支持部署独立 Ceph 集群，也支持接入外部已部署的 Ceph 集群（External Cluster 模式）。

然而，Rook 存在以下局限性：

* Ceph 在高负载或恢复场景下对底层硬件性能和网络延迟敏感，Rook 依赖于 Kubernetes 集群本身的稳定性，故障排查链路较长，极端场景下可能需要额外考虑 K8s 调度、网络、Pod 生命周期等多个维度
* Rook 设计上更适合一个 K8s 集群部署一套独立的 Ceph 集群，虽然解耦性好，但是存储资源在集群间难以服用，资源成本和运维复杂度都不小
* Rook 只适合云原生场景，不兼容裸金属、私有云、公有云等混合架构下，难以作为统一管理的底层存储平台

本系列将采用 cephadm 部署 Ceph 集群，Rook 作为消费端接入的架构，兼顾了运维可控性与云原生生态的集成能力，适用于混合架构下的统一存储场景。对于 Dev/Test/Staging 等非生产环境，可共用一套由 cephadm 部署的中小型集群；对于 Prod 环境，建议独立部署一套 Ceph 集群。总的来说，这种方式可以很好的适配混合架构，避免 Kubernetes 本身的不稳定影响，并降低重复部署成本。

## Ceph 作为统一存储平台

Ceph 支持块（RBD）、文件（CephFS）、对象（RGW）等多种存储协议：

| 协议类型        | 场景                                |
| --------------- | ----------------------------------- |
| RBD 块存储      | MySQL、PostgreSQL、MongoDB 等数据库 |
| CephFS 文件存储 | 共享挂载、日志存储、NFS 服务        |
| RGW 对象存储    | S3、备份、AI 模型管理               |

Ceph 作为统一底层存储平台是一个优秀方案，但对于对 I/O 延迟极其敏感的存储场景（如 etcd、Redis 等），其表现可能不足，此时建议考虑以下存储策略：

* Local PV + StatefulSet：通过配置节点亲和性并使用节点本地硬盘（SSD/HDD），适用于对性能敏感的服务，提供高 IOPS 和低延迟
* 外部部署或云存储服务：追求 K8s 集群无状态，将存储服务与 K8s 完全解耦，Kubernetes 集群变动、迁移、升级时，存储服务可实现无感知

## 部署前准备

下面我们计划使用 cephadm 在实验环境构建一个三节点的 Ceph 集群，部署前应完成一些准备工作。

核心组件版本：

```yaml
OS: Debian GNU/Linux 12（bookworm）
Podman: 4.3.1
Ceph: 19.2.2（squid）
Kubernetes: 1.31.7
Rook: v1.17.0
```

部署前确认以下事项：

* 三台裸机或虚拟机（这里以 `192.168.0.{150..152}` 三台为例）
* 每个节点至少为 Ceph 准备一块数据盘（无需格式化）
* 操作系统支持 Python3、Systemd、Podman/Docker、LVM2（目前主流操作系统基本都支持）
* 已进行时间同步

📌注：强烈推荐使用 Podman 代替 Docker，Podman 支持 rootless 模式、无守护进程（daemonless）运行，且可原生集成 systemd 单元服务，系统亲和性更强。由于 Red Hat 同时是 Ceph 与 Podman 的主要维护方，使用 Podman 能获得更好的兼容性、安全性与官方支持保障。

## 部署一个新集群

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

1. 引导创建新集群

    ```bash
    cephadm bootstrap --mon-ip 192.168.0.150
    ```

    该条命令做了以下几件事：

    * 在本地主机上创建 MON 和 MGR 守护进程
    * 生成集群密钥
    * 配置了最小化的 ceph.conf 文件及 admin 密钥文件
    * 将当前主机打上 _admin 标签以便管理访问

    这里额外给出几个常用的参数，可执行 `cephadm bootstrap -h` 查看更多参数：

    * `--allow-fqdn-hostname`：允许使用 FQDN 作为主机名
    * `--log-to-file`：输出日志到本地文件
    * `--public-network`：指定客户端与 MON、OSD、MDS、RGW 等 Ceph 组件通信的网络
    * `--cluster-network`：指定 OSD 节点间专用的内部网络，用于数据复制、恢复和心跳检测
    * `--config`：指定自定义的配置文件路径

1. 启用 ceph cli

    要执行 ceph 命令，需要通过 ceph shell 的方式到容器中运行：

    ```bash
    cephadm shell -- ceph -s
    ```

    为了方便运维，建议安装 ceph-common 软件包，里面包含了所有 ceph 命令，包括 ceph、rbd、mount.ceph（用于挂载CephFS文件系统）等：

    ```bash
    cephadm install ceph-common
    ```

    接下来，便可以直接使用 ceph 命令了，比如查看 ceph 集群状态：

    ```bash
    ceph status
    ```

    应该会看到一个 mon 和 一个 mgr 进程：

    ```bash
    cluster:
        id:     852837fa-46a1-11f0-a6ed-00505639e7ae
        health: HEALTH_WARN
                OSD count 0 < osd_pool_default_size 3
    
    services:
        mon: 1 daemons, quorum kube1 (age 2m)
        mgr: kube1.iuiljm(active, since 54s)
        osd: 0 osds: 0 up, 0 in
    
    data:
        pools:   0 pools, 0 pgs
        objects: 0 objects, 0 B
        usage:   0 B used, 0 B / 0 B avail
        pgs:   
    ```

1. 添加其他主机到集群中：

    将集群的公共 SSH 密钥安装到新主机 root 用户的 authorized_keys 文件中：
  
    ```bash
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@kube2
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@kube3
    ```

    通知 Ceph 有新节点加入集群：

    ```bash
    ceph orch host add kube2 192.168.0.151 --labels _admin
    ceph orch host add kube3 192.168.0.152 --labels _admin
    ```

    📌注：默认情况下，_admin 标签会让 cephadm 在该主机的 /etc/ceph 目录下维护一份 ceph.conf 配置文件和 client.admin 密钥环文件，适用于部署 MON、MGR 等关键服务节点。

    可以通过此命令查看各节点服务列表：

    ```bash
    ceph orch ls
    ```

    如果发现输出结果有点不符预期，那是因为 `cephadm boostrap` 默认策略下 mon 数量为 5，由于我这里只有 3 个节点，所以需要手动将 mon 数量调整成 3：

    ```bash
    ceph orch apply mon count:3
    ```

    输出结果如下：

    ```bash
    NAME           PORTS        RUNNING  REFRESHED  AGE  PLACEMENT  
    alertmanager   ?:9093,9094      1/1  30s ago    5m   count:1    
    ceph-exporter                   3/3  30s ago    5m   *          
    crash                           3/3  30s ago    6m   *          
    grafana        ?:3000           1/1  30s ago    5m   count:1    
    mgr                             2/2  30s ago    6m   count:2    
    mon                             3/3  30s ago    1s   count:3    
    node-exporter  ?:9100           3/3  30s ago    5m   *          
    prometheus     ?:9095           1/1  30s ago    5m   count:1    
    ```

    📌注：这里为了方便这里的 placement 采用了 `count:[num]` 方式进行随机分配，但在实际生产环境中，强烈推荐先使用 `ceph orch host label add <hostname> <label>` 为主机上各服务预先打好标签，以实现更精确、可控的部署策略。

## 添加 OSD 存储

1. 检查所有集群主机上的存储设备信息：

    ```bash
    ceph orch device ls
    ```

    存储设备可用的前提是：
    * 设备上不能有任何分区
    * 设备上不能存在任何 LVM 状态
    * 设备不能被挂载
    * 设备上不能包含文件系统
    * 设备上不能包含 Ceph BlueStore OSD
    * 设备容量必须大于 5 GB
    * Ceph 不会在不符合条件的设备上创建 OSD

1. 执行以下命令以预演配置变更（可能需要执行两次也会显示结果）：

    ```bash
    ceph orch apply osd --all-available-devices --dry-run
    ```

    📌注：`ceph orch apply` 会让 cephadm 持续对状态进行协调（reconcile），保证集群状态与期望配置一致。
    换句话说，Ceph 集群会持续监控和自动管理符合条件的设备：
    * 新加入集群的磁盘会被自动发现并用于创建新的 OSD
    * 如果某个 OSD 被移除且对应的 LVM 物理卷被清理（zap），Ceph 也会自动在该设备上重新创建新的 OSD

1. 继续执行完成 osd 创建，敏感环境中可以追加 `--unmanaged` 参数改用手动管理以避免误操作：

    ```bash
    ceph orch apply osd --all-available-devices
    ```

1. 或者你也可以手动指定，这样更安全可控（可选）

    ```bash
    ceph orch daemon add osd *<host>*:*<device-path>*
    ```

1. 稍微一会，检查集群状态，显示已有 3 个 osd 加入到集群中，且为 up 状态：

    ```bash
    cluster:
        id:     852837fa-46a1-11f0-a6ed-00505639e7ae
        health: HEALTH_OK
    
    services:
        mon: 3 daemons, quorum kube1,kube2,kube3 (age 22m)
        mgr: kube1.iuiljm(active, since 24m), standbys: kube2.mpdvnm
        osd: 3 osds: 3 up (since 68s), 3 in (since 2m)
    
    data:
        pools:   1 pools, 1 pgs
        objects: 2 objects, 449 KiB
        usage:   81 MiB used, 30 GiB / 30 GiB avail
        pgs:     1 active+clean  
    ```

1. 内存自动调节优化项

    cephadm 引导集群时默认会启用 `osd_memory_target_autotune = true`，并设置 `mgr/cephadm/autotune_memory_target_ratio = 0.7`。也就是说，每台主机的 OSD 默认最多使用该节点总内存的 70%，用于 BlueStore 的缓存，当你的存储集群上有其他服务时，可建议按需调小该参数，防止 OOM。

    ```bash
    ceph config set osd osd_memory_target_autotune true
    ceph config set mgr mgr/cephadm/autotune_memory_target_ratio 0.3
    ```

## 集群的卸载与清除

使用 cephadm 卸载集群非常简单，只需执行以下几个命令。

1. 停止集群编排功能

    cephadm 默认启用了集群编排（orchestration）功能，为避免执行过程中有新组件被重新部署，必须先禁用 cephadm 管理模块：

    ```bash
    ceph mgr module disable cephadm
    ```

1. 确认集群 FSID

    ```bash
    ceph fsid
    ```

    示例输出如下：

    ```bash
    1ec2ce44-1fff-11f0-a457-00505639e7ae
    ```

1. 在集群的*每一台主机上*，执行以下命令清除所有 ceph 服务、数据盘及元数据：

    ```bash
    cephadm rm-cluster --force --zap-osds --fsid 1ec2ce44-1fff-11f0-a457-00505639e7ae
    ```

    参数说明：
   * `--force`：强制移除，不做额外确认
   * `--zap-osds`：清空并格式化所有已部署的 OSD 设备（彻底销毁数据）
   * `--fsid`：指定要移除的集群唯一 ID

1. 进一步执行以下操作清理残留文件和依赖：

    ```bash
    rm -rf /etc/ceph/* /var/lib/ceph/* /var/log/ceph/*
    ```

至此，关于 cephadm 的部分已简单介绍完毕，可以点击继续探索 [Ceph: 02-CephFS 与 NFS Gateway 部署实践]({{< relref "posts/infra/storage/02-cephfs.md" >}})。
