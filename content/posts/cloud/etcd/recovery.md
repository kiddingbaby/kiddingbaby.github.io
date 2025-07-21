---
title: "Etcd 集群整体宕机的恢复方案"
date: 2025-01-01T09:00:00+08:00
lastmod: 2025-06-05T12:00:00+08:00
description: "当 etcd 高可用集群全部节点宕机时，如何通过快照恢复、节点替换与数据一致性检查，完成 etcd 的灾难恢复。"
categories: ["云平台"]
tags: ["etcd", "灾备", "Kubernetes"]
series: ["etcd"]
keywords: ["etcd 故障恢复", "Kubernetes etcd 恢复", "etcd snapshot restore", "etcd 集群整体宕机"]
ShowToc: true
TocOpen: true
draft: false
---
 
## 背景

Kubernetes 集群目前并没有提供关机重启的选项，因此维护 etcd 集群的稳定至关重要，在生产环境下推荐以 External 方式来部署 etcd 集群，并放在单独的区域内。

Kubespray 默认使用 host 模式来部署外部 etcd 集群，只需依赖 systemd 即可，这是生产下经过长期验证的部署方式。kubeadm 官方建议通过 static pod 方式来部署外部 etcd 集群，这样的优势是在云原生环境下更具一致性，但缺点是额外增加了 kubelet、CRI 等依赖，在已有 Kubespray 支持的前提下，个人推荐直接使用 host 模式来部署外部 etcd。

但是天有不测风云，总是有可能会遇到服务器整体宕机，从而导致 etcd 集群无法启动的情况。那么，如何进行灾难恢复（Disaster Recovery）呢？

本文将通过一个现实的案例，给出一个生产下可行的 etcd 集群整体宕机后的恢复方案。

## 实践案例

核心组件版本：

```yaml
Kubernetes: 1.31.7
etcd: 3.5.19
```

在我的实验环境中，[使用 Kubespray 在虚拟化环境中搭建一个 3 节点的 HA Kubernetes 集群]({{< relref "posts/cloud/k8s/02-kubespray.md" >}})，其中外部 etcd 使用 systemd 管理。但由于频繁的关机、休眠，非常容易出现 etcd 集群整体不可用、Raft 状态错误或 WAL 损坏等情况。

这里我们要分两种情况：

### 第一种：etcd 集群还能正常运行

针对这种就比较好办，可以从健康节点导出快照，再对故障节点执行 Member Recovery 即可，此处不做赘述。

### 第二种：所有节点 etcd 均无法正常运行

这时要做以下几件事。

1. 确认所有 etcd 服务关闭，并关闭 kubelet 和 kube-apiserver 服务避免干扰。

    ```bash
    systemctl stop etcd
    systemctl stop kubelet
    systemctl stop kube-apiserver（如果是 systemd 管理）
    ```

1. 在 每个节点 执行以下命令查看快照信息。

    ```bash
    etcdutl snapshot status /var/lib/etcd/member/snap/db
    ```

    输出结果：

    ```bash
    334118b6, 1768088, 3563, 18 MB
    ```

    上面的输出分别表示：快照的哈希值、快照对应的 etcd revision（版本号）、快照中存储的 key 数量、快照文件大小。我们应选择 revision 最大的快照来执行恢复，最大程度减少数据损失。

1. 备份原始 etcd 数据并清空数据目录。

    ```bash
    mkdir -p /var/lib/etcd.bak
    mv /var/lib/etcd/* /var/lib/etcd.bak/
    ```

1. 选定快照后，将该快照文件同步到其他机器对应目录下，并 **分别在每个节点** 执行以下命令执行恢复：

    ```bash
    etcdutl snapshot restore /var/lib/etcd.bak/member/snap/db --data-dir /var/lib/etcd --name <当前节点的etcd的名称标识> \
        --initial-cluster etcd1=https://192.168.0.150:2380,etcd2=https://192.168.0.151:2380,etcd3=https://192.168.0.152:2380 \
        --initial-advertise-peer-urls https://<当前节点的IP地址>:2380 \
        --skip-hash-check=true
    ```

    执行 `etcdutl snapshot restore` 后，通常会：

    * 重建全新的 `--data-dir`，包括新的 `wal/` 和 `snap/` 目录
    * 解析指定的快照文件，重建数据
    * 按照 `--name`，`--initial-cluster`，`--initial-advertise-peer-urls` 参数生成新的 Raft 集群配置（确保参数和之前一致）

1. 接下来通过 systemd 重启服务。

    ```bash
    systemctl start etcd
    ```

1. 检查集群健康状态后，并顺序恢复 kubelet、kube-apiserver 等服务。

    ```bash
    etcdctl --endpoints=https://192.168.0.150:2379,https://192.168.0.151:2379,https://192.168.0.152:2379 \
    --cacert=/etc/ssl/etcd/ssl/ca.pem \
    --cert=/etc/ssl/etcd/ssl/admin-kube3.pem \
    --key=/etc/ssl/etcd/ssl/admin-kube3-key.pem \
    endpoint health
    ```

    输出状态：

    ```bash
    https://192.168.0.150:2379 is healthy: successfully committed proposal: took = 17.176498ms
    https://192.168.0.152:2379 is healthy: successfully committed proposal: took = 18.531198ms
    https://192.168.0.151:2379 is healthy: successfully committed proposal: took = 19.581746ms
    ```

1. 顺序恢复其他服务即可。

    ```bash
    systemctl start kubelet
    systemctl start kube-apiserver
    ```
