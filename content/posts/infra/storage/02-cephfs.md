---
title: "Ceph: 02-CephFS 与 NFS Gateway 部署实践"
date: 2025-06-12T10:00:00+08:00
lastmod: 2025-06-12T10:00:00+08:00
categories: ["基础设施"]
tags: ["CephFS", "NFS", "分布式文件系统", "高可用", "企业实践"]
series: ["Ceph"]
keywords: ["CephFS 部署", "NFS Ganesha", "分布式文件存储"]
ShowToc: true
TocOpen: true
draft: true
---

## CephFS 是什么

本文接上篇 [基于 cephadm 部署高可用 Ceph 存储集群]({{< relref "posts/infra/storage/01-cephadm.md" >}})。

Ceph 文件系统 ( CephFS ) 是一个符合 POSIX 标准的文件系统，它构建于 Ceph 的分布式对象存储RADOS之上。

![CephFS 架构图](/images/posts/cephfs-architecture.svg)

之前我们已经部署好了一个简单的 Ceph 集群，并添加了 OSD。话不多说，接下来我们继续。

## 部署 CephFS

要使用 CephFS 文件系统，需要一个或多个 MDS（Metadata Server）守护进程，这里使用较新的 `ceph fs volume` 接口来创建新的文件系统，并通过 `ceph orch` 来配置 MDS。

1. 创建 CephFS 卷

    ```bash
    ceph fs volume create cephfs
    ```

    * `cephfs`：为 CephFS 文件系统指定的名称

1. 利用 Ceph Orchestrator 为文件系统自动创建并配置 MDS

    ```bash
    ceph orch apply mds cephfs 'count:3'
    ```

1. 验证 cephfs 状态

    ```bash
    ceph fs status
    ```

    默认情况下会有一台为 active，其余节点均为 standby：

    ```bash
    cephfs - 0 clients
    ======
    RANK  STATE           MDS             ACTIVITY     DNS    INOS   DIRS   CAPS  
    0    active  cephfs.kube2.tekede  Reqs:    0 /s    10     13     12      0   
        POOL           TYPE     USED  AVAIL  
    cephfs.cephfs.meta  metadata  96.0k  9697M  
    cephfs.cephfs.data    data       0   9697M  
        STANDBY MDS      
    cephfs.kube1.kjracz  
    cephfs.kube3.rdjvxy  
    MDS version: ceph version 19.2.2 (0eceb0defba60152a8182f7bd87d164b639885b8) squid (stable)
    ```

## 部署 NFS

中小企业内部通常通过服务器搭配 NFS 或 SMB 协议实现文件共享。由于 Ceph 对 SMB 协议的支持尚处于开发阶段，功能尚不完善，建议在 Ceph 集群中部署 NFS Ganesha 服务，将 CephFS 目录以 NFS 协议导出供客户端访问。通过这种方式，Ceph 能作为统一的分布式存储平台，稳定高效地提供 NFS 文件共享，满足企业内网多终端的访问需求。

📌注：目前仅支持 NFSv4 协议。

1. 由于存储层是 RADOS，天然具备高可用性，如需实现访问层的高可用可参考官方文档配置 ingress 和 haproxy ，本文采用了单节点方式进行演示：

    ```bash
    ceph orch apply nfs nfs
    ```

    * `nfs`：第二个 nfs 指的是 <svc_id>，可按需命名

1. 验证 NFS Ganesha 状态：

    ```bash
        ceph orch ls --service_type=nfs
    ```

    输出如下：

    ```bash
    NAME     PORTS  RUNNING  REFRESHED  AGE  PLACEMENT  
    nfs.nfs             1/1  3m ago     3m   count:1    
    ```

1. 在 CephFS 中创建 subvolume，供 NFS 测试：

    ```bash
    ceph fs subvolume create cephfs nfsdata
    ```

1. 可通过以下命令查看具体的 path：

    ```bash
    ceph fs subvolume getpath cephfs nfsdata
    ```

    结果显示：

    ```bash
    /volumes/_nogroup/nfsdata/8eb9327b-e923-4923-9bd9-cab893f95435/
    ```

1. 利用 NFS Ganesha 导出 CephFS：

    ```bash
    ceph nfs export create cephfs --cluster-id nfs --pseudo-path /nfsdata --fsname cephfs --path /volumes/_nogroup/nfsdata/8eb9327b-e923-4923-9bd9-cab893f95435/
    ```

1. 验证是否导出成功：

    ```bash
    ceph nfs export ls nfs --detailed 
    ```

    检查是否有 "cluster_id" 为 "nfs" 的数据条目：

    ```bash
    [
        {
            "access_type": "RW",
            "clients": [],
            "cluster_id": "nfs",
            "export_id": 1,
            "fsal": {
            "cmount_path": "/",
            "fs_name": "cephfs",
            "name": "CEPH",
            "user_id": "nfs.nfs.cephfs.405e1f9f"
            },
            "path": "/volumes/_nogroup/nfsdata/8eb9327b-e923-4923-9bd9-cab893f95435/",
            "protocols": [
                4
            ],
            "pseudo": "/nfsdata",
            "security_label": true,
            "squash": "none",
            "transports": [
            "TCP"
            ]
        }
    ]
    ```

1. 客户端安装 NFS 软件后，完成挂载：

    ```bash
    mount -t nfs4 -o nfsvers=4 192.168.0.150:/nfsdata /mnt
    ```

1. 接下来便可进入 `/mnt` 完成读写操作，也可以在多个客户端挂载验证，比如我们可以测试写入一个 1GiB 的文件：

    ```bash
    dd if=/dev/zero of=/mnt/nfsdata/8eb9327b-e923-4923-9bd9-cab893f95435/test-1GiB.bin bs=1M count=1024 status=progress
    ```

1. 执行 `ceph fs status` 查看状态，可以看到 CephFS 的 data pool 中已写入 `1024M * 3` 的数据：

    ```bash
    cephfs - 1 clients
    ======
    RANK  STATE           MDS             ACTIVITY     DNS    INOS   DIRS   CAPS  
    0    active  cephfs.kube2.tekede  Reqs:    0 /s    23     21     16      8   
        POOL           TYPE     USED  AVAIL  
    cephfs.cephfs.meta  metadata  1119k  8646M  
    cephfs.cephfs.data    data    3072M  8646M  
        STANDBY MDS      
    cephfs.kube1.kjracz  
    cephfs.kube3.rdjvxy  
    MDS version: ceph version 19.2.2 (0eceb0defba60152a8182f7bd87d164b639885b8) squid (stable)
    ```

接下来，我们可以继续完成 [Ceph: 03-RGW 部署与 S3 兼容接口测试]({{< relref "posts/infra/storage/03-rgw.md" >}})。
