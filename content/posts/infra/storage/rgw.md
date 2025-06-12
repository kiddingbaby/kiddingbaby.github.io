---
title: "基于 cephadm 部署高可用 Ceph 存储集群"
date: 2025-06-12T10:00:00+08:00
lastmod: 2025-06-12T10:00:00+08:00
description: "通过 cephadm 工具部署企业级高可用 Ceph 集群，支持多节点管理、自动化运维、Dashboard UI，适用于生产环境。"
categories: ["基础设施"]
tags: ["Ceph", "cephadm", "高可用", "分布式存储", "企业实践"]
series: ["部署方案"]
keywords: ["cephadm 高可用", "Ceph 存储集群", "Ceph 自动化部署"]
ShowToc: true
TocOpen: true
draft: true
---

## 部署工具选型

Ceph 是一款企业级开源分布式存储系统，具备高可扩展性与强一致性。早期 Ceph 部署主要采用 ceph-deploy 配合 Ansible 的方式，目前，ceph-deploy 已逐步被淘汰，而 ceph-ansible 项目自 2022 年 8 月起停止发布新版本，社区建议迁移至 cephadm。从 Octopus（15.2.x）版本起，Ceph 官方引入并推荐使用基于容器的部署与运维工具 cephadm，用于统一管理集群的部署、升级与服务编排。

在 Kubernetes 环境中，Rook 提供原生方式来消费 Ceph 存储，是当前最主流的集成方案。Rook 支持部署独立 Ceph 集群，也支持接入外部已部署的 Ceph 集群（External Cluster 模式）。

然而，Rook 存在以下局限性：

* Ceph 在高负载或恢复场景下对底层硬件性能和网络延迟敏感，Rook 依赖于 Kubernetes 集群本身的稳定性，故障排查链路较长，极端场景下可能需要额外考虑 K8s 调度、网络、Pod 生命周期等多个维度
* Rook 设计上更适合一个 K8s 集群部署一套独立的 Ceph 集群，虽然解耦性好，但是存储资源在集群间难以服用，资源成本和运维复杂度都不小
* Rook 只适合云原生场景，不兼容裸金属、私有云、公有云等混合架构下，难以作为统一管理的底层存储平台

本文采用 cephadm 部署 Ceph 集群，Rook 作为消费端接入的架构，兼顾了运维可控性与云原生生态的集成能力，适用于混合架构下的统一存储场景。对于 Dev/Test/Staging 等非生产环境，可共用一套由 cephadm 部署的中小型集群；对于 Prod 环境，建议独立部署一套 Ceph 集群。总的来说，这种方式可以很好的适配混合架构，避免 Kubernetes 本身的不稳定影响，并降低重复部署成本。

## Ceph 作为统一存储平台

Ceph 支持块（RBD）、文件（CephFS）、对象（RGW）等多种存储协议：

| 协议类型        | 场景                                |
| --------------- | ----------------------------------- |
| RBD 块存储      | MySQL、PostgreSQL、MongoDB 等数据库 |
| CephFS 文件存储 | 共享挂载、日志存储、NFS 服务        |
| RGW 对象存储    | S3、备份、AI 模型管理               |

Ceph 作为统一底层存储平台是一个优秀方案，但对于对 I/O 延迟极其敏感的存储场景（如 etcd、Redis 等），其表现可能不足，此时建议考虑以下存储策略：

* Local PV + StatefulSet：通过配置节点亲和性并使用节点本地硬盘（SSD/HDD），适用于对性能敏感的服务，提供高 IOPS 和低延迟
* 外部部署或云存储服务：追求 K8s 集群无状态，将存储服务与 K8s 完全解耦，Kubernetes 集群变动、迁移、升级时，存储服务可实现无感知，提高可维护性和弹性

## 部署高可用的 ceph 集群

下面我们将使用 cephadm 在来构建一个三节点的高可用 Ceph 集群。

核心组件版本：

```yaml
OS: Debian GNU/Linux 12（bookworm）
Podman: 4.3.1
Ceph: 19.2.2（squid）
Kubernetes: 1.31.7
Rook: v1.17.0
```

📌注：强烈推荐使用 Podman 代替 Docker，Podman 支持 rootless 模式、无守护进程（daemonless）运行，且可原生集成 systemd 单元服务，系统亲和性更强。由于 Red Hat 同时是 Ceph 与 Podman 的主要维护方，使用 Podman 能获得更好的兼容性、安全性与官方支持保障。

部署前确认以下事项：

* 三台裸机或虚拟机（这里以 192.168.0.{150..152} 三台为例）
* 每个节点至少为 Ceph 准备一块数据盘（无需格式化）
* 操作系统支持 Python3、Systemd、Podman/Docker、LVM2（目前主流操作系统基本都支持）
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

1. 添加 OSD 存储：

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

1. 开启内存自动调节选项

    cephadm 引导集群时默认会启用 `osd_memory_target_autotune = true`，并设置 `mgr/cephadm/autotune_memory_target_ratio = 0.7`。也就是说，每台主机的 OSD 默认最多使用该节点总内存的 70%，用于 BlueStore 的缓存，当你的存储集群上有其他服务时，可建议按需调小该参数，防止 OOM。

    ```bash
    ceph config set osd osd_memory_target_autotune true
    ceph config set mgr mgr/cephadm/autotune_memory_target_ratio 0.3
    ```

1. 开启 dashboard

    1. 通过以下命令启用 dashboard：

    ```bash
    ceph mgr module enable dashboard
    ```

    1. 设置 SSL/TLS 支持，这里仅使用自签名证书：

    ```bash
    ceph dashboard create-self-signed-cert
    ```

    1. 创建一个管理员用户 admin，这里使用 `--force-password` 绕过密码策略检查，并使用 `--pwd_update_required` 确保首次登录后强制要求用户更改密码：

    ```bash
    echo "123456" > admin.pass
    ceph dashboard ac-user-create --force-password --pwd_update_required admin -i admin.pass administrator
    ```

    1. 目前我们的集群里有两个 mgr，为了提高可用性，我们需要引入代理服务，建议先执行以下命令：

    ```bash
    ceph config set mgr mgr/dashboard/standby_behaviour "error"
    ceph config set mgr mgr/dashboard/standby_error_status_code 503
    ```

    📌注：上述命令会让非 active 的 mgr dashboard 不响应 303 Redirect，而是自定义相应 503 状态码，否则代理服务在 dashboard failover 后将收到错误跳转地址。

    1. 官方给的是 haproxy 示例，我的环境仅部署了 openresty，因此这里给出 openresty 的基本配置，如果两个节点以上建议引入 health check 实现动态 upstream：

    ```bash
    upstream ceph-dashboard-backend {
        server 192.168.0.150:8443;
        server 192.168.0.151:8443;
    }

    server {
        listen 443 ssl;
        server_name ceph-dashboard.example.internal;

        include /etc/openresty/conf.d/ssl.conf;

        access_log /var/log/openresty/ceph-dashboard-access.log;
        error_log  /var/log/openresty/ceph-dashboard-error.log;

        location / {
            proxy_pass https://ceph-dashboard-backend;

            proxy_ssl_server_name on;
            proxy_ssl_verify off;

            proxy_connect_timeout 5s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_next_upstream error timeout http_503;
        }
    }
    ```

1. 添加 CephFS

    ![CephFS 架构图](/images/posts/cephfs-architecture.svg)

    Ceph 文件系统 ( CephFS ) 是一个符合 POSIX 标准的文件系统，它构建于 Ceph 的分布式对象存储RADOS之上。

    要使用 CephFS 文件系统，需要一个或多个 MDS（Metadata Server）守护进程。如果使用较新的 `ceph fs volume` 接口来创建新的文件系统，这些守护进程会自动创建。

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

1. 添加 NFS（可选）

    中小企业内部通常通过服务器搭配 NFS 或 SMB 协议实现文件共享。由于 Ceph 对 SMB 协议的支持尚处于开发阶段，功能尚不完善，建议在 Ceph 集群中部署 NFS Ganesha 服务，将 CephFS 目录以 NFS 协议导出供客户端访问。通过这种方式，Ceph 能作为统一的分布式存储平台，稳定高效地提供 NFS 文件共享，满足企业内网多终端的访问需求。

    📌注：目前仅支持 NFSv4 协议。

    1. 由于存储层是 RADOS，天然具备高可用性，如果需要可以参考官方文档配置 ingress 和 haproxy 实现访问层的高可用，本文采用了单节点方式进行演示：

        ```bash
        ceph orch apply nfs nfs
        ```

    1. 验证 NFS Ganesha 状态：

        ```bash
            ceph orch ls --service_type=nfs
        ```

        输出如下：

        ```bash
        NAME     PORTS  RUNNING  REFRESHED  AGE  PLACEMENT  
        nfs.nfs             1/1  3m ago     3m   count:1    
        ```

    1. 在 CephFS 中创建 subvolume，专供 NFS 测试：

        ```bash
        ceph fs subvolume create cephfs nfsdata
        ```

    1. 可通过以下命令查看具体的 path：

        ```bash
        ceph fs subvolume getpath cephfs nfsdata
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

1. 添加 RGW

    RGW（RADOS Gateway）是 Ceph 提供的对象存储网关服务，基于 HTTP 协议，实现了对 Ceph RADOS 分布式对象存储的访问接口，兼容了 Amazon S3 和 OpenStack Swift 接口，可部署在单集群或多站点模式下，适用于私有云与混合云环境。
    中小企业通常单 Realm + 单 Zone 即可满足需求，此处部署了 3 个 RGW 实例，提供 S3 兼容的对象存储服务，并预留了基于多站点（Multisite）架构的扩展能力。。

    1. 使用 Multisite 架构初始化：

        ```bash
        # 初始化一个 Realm
        radosgw-admin realm create --rgw-realm=default --default

        # 创建一个 ZoneGroup，设为默认且为主 ZoneGroup
        radosgw-admin zonegroup create --rgw-zonegroup=cn --rgw-realm=default --endpoints=https://ceph-rgw.example.internal --master --default

        # 创建一个 Zone，设为默认且为主 Zone
        radosgw-admin zone create --rgw-zonegroup=cn --rgw-zone=cn-hangzhou --endpoints=https://ceph-rgw.example.internal --master --default

        # 创建一个系统同步用户，记录生成的 access_key 和 secret_key
        radosgw-admin user create --uid="synchronization-user" --display-name="Synchronization User" --system

        # 将系统用户添加到主 Zone
        radosgw-admin zone modify --rgw-zone=cn-hangzhou --access-key="ZRKBG2ODB20TPPR1X9GU" --secret="KErluNUPfiXR8LnZynePMDfyN6JLQG2sihNIzxnZ"

        # 提交 Period 配置
        radosgw-admin period update --commit
        ```

    1. 部署 RGW：

        ```bash
        ceph orch apply rgw rgw --realm=default --zonegroup=cn --zone=cn-hangzhou --placement="count:3" --port=8000
        ```

    1. Openresty 代理配置如下：

        ```bash
        upstream ceph-rgw-backend {
            server 192.168.0.150:8000;
            server 192.168.0.151:8000;
            server 192.168.0.152:8000;
        }

        server {
            listen 443 ssl;
            server_name ceph-rgw.example.internal;

            include /etc/openresty/conf.d/ssl.conf;

            access_log      /var/log/openresty/ceph-rgw-access.log;
            error_log       /var/log/openresty/ceph-rgw-error.log;

            location / {
                client_max_body_size 10G;
                proxy_pass http://ceph-rgw-backend;

                proxy_connect_timeout 5s;
                proxy_send_timeout 30s;
                proxy_read_timeout 30s;

                proxy_set_header Connection $http_connection;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;

                proxy_next_upstream error timeout http_503;
            }
        }
        ```

    1. 启用 mgr 中的 rgw 模块：

        ```bash
        ceph mgr module enable rgw
        ```

    1. 创建一个管理员用户用于 S3 接入：

        ```bash
        radosgw-admin user create --uid=admin --display-name="RGW Admin User" --email=admin@example.internal --admin
        ```

    1. 在客户端安装并配置 s3cmd 工具，完成对象存储的测试数据上传：

        ```bash
        s3cmd mb s3://test
        dd if=/dev/zero of=./test-64M.bin bs=1M count=64 status=progress
        s3cmd put test-64M.bin s3://test/
        ```

        上传成功后，执行以下命令：

        ```bash
        s3cmd info s3://test/test-64M.bin
        ```

        s3cmd 会返回如下对象元数据信息：

        ```bash
        s3://test/test-64M.bin (object):
        File size: 67108864
        Last mod:  Thu, 12 Jun 2025 16:13:35 GMT
        MIME type: application/octet-stream
        Storage:   STANDARD
        MD5 sum:   7f614da9329cd3aebf59b91aadc30bf0
        SSE:       none
        Policy:    none
        CORS:      none
        ACL:       RGW Admin User: FULL_CONTROL
        x-amz-meta-s3cmd-attrs: atime:1749744690/ctime:1749744535/gid:0/gname:root/md5:7f614da9329cd3aebf59b91aadc30bf0/mode:33188/mtime:1749744535/uid:0/uname:root
        ```

## ceph 集群的卸载与清除

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
