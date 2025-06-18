---
title: "Ceph: 04-MGR Dashboard 部署与实践"
date: 2025-06-12T10:00:00+08:00
lastmod: 2025-06-12T10:00:00+08:00
categories: ["基础设施"]
tags: ["Ceph", "MGR Dashboard", "运维管理", "高可用", "企业实践"]
series: ["Ceph"]
keywords: ["Ceph Dashboard 部署", "Ceph MGR Web 界面", "Ceph 运维监控"]
ShowToc: true
TocOpen: true
draft: false
---

## 启用 MGR Dashboard

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

📌注：上述命令会让非 Active 的 MGR dashboard 不响应 303 Redirect，而是自定义相应 503 状态码，否则代理服务在 dashboard failover 后将收到错误跳转地址。

1. 本人实验环境仅部署了轻量反向代理，这里给出 Openresty/Nginx 的基本配置，建议引入支持主动 Health Check 的网关实现动态 upstream，比如 HAProxy、Envoy、APISIX：

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

1. 访问正常后，我们还需启用 MGR 中的 RGW 模块：

```bash
ceph mgr module enable rgw
```
