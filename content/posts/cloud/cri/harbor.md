---
title: "镜像仓库"
date: 2025-06-14T10:00:00+08:00
lastmod: 2025-06-14T10:00:00+08:00
categories: [""]
tags: [""]
series: [""]
keywords: [""]
ShowToc: true
TocOpen: true
draft: true
---

Harbor

## CI 机器人

## 镜像加速

网络方案：Nginx 反代 Docker 仓库和 auth.docker.io + Cloudflare Workers + 透明代理

通过 ratelimit-remaining 值查看剩余拉取次数：

```bash
TOKEN=$(curl "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)
curl --head -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest
```

## 代理缓存

## 镜像漏洞扫描

## 安全加固
