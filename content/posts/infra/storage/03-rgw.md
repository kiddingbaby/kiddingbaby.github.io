---
title: "Ceph: 03-RGW 部署与 S3 兼容接口测试"
date: 2025-06-12T10:00:00+08:00
lastmod: 2025-06-12T10:00:00+08:00
categories: ["基础设施"]
tags: ["RGW", "对象存储", "S3", "cephadm", "高可用", "企业实践"]
series: ["Ceph"]
keywords: ["Ceph RGW 部署", "S3 兼容存储", "cephadm 对象存储"]
ShowToc: true
TocOpen: true
draft: false
---

## RGW 是什么

本文接上篇 [Ceph: 02-CephFS 与 NFS Gateway 部署实践]({{< relref "posts/infra/storage/02-cephfs.md" >}})。

RGW（RADOS Gateway）是 Ceph 提供的对象存储网关服务，基于 HTTP 协议，实现了对 Ceph RADOS 分布式对象存储的访问接口，兼容了 Amazon S3 和 OpenStack Swift 接口，可部署在单集群或多站点模式下，适用于私有云与混合云环境。

本篇将介绍如何部署 RGW 服务，为集群提供对象存储能力，并通过 s3cmd 进行一个简单的测试。

## 部署 RGW

中小企业通常单 Realm + 单 Zone 即可满足需求，本文将部署了 3 个 RGW 实例，提供 S3 兼容的对象存储服务，并预留基于多站点（Multisite）架构的扩展能力。

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

## s3cmd 验证

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

1. 上传成功后，执行以下命令：

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

继续下一篇：[Ceph: 04-MGR Dashboard 部署与企业级实践]({{< relref "posts/infra/storage/04-dashboard.md" >}})。
