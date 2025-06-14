---
title: "大规模集群下 Ansible 的关键配置与优化策略"
date: 2025-06-14T10:00:00+08:00
lastmod: 2025-06-14T10:00:00+08:00
categories: ["运维自动化", "DevOps"]
tags: ["Ansible", "大规模集群", "自动化运维", "高可用", "配置管理", "失败重试"]
series: ["Ansible"]
keywords: ["Ansible", "Ansible 失败重试", "自动化运维优化", "集群管理"]
ShowToc: true
TocOpen: true
draft: false
---

Ansible 在现代企业中，尤其是私有云、裸金属环境下是非常流行的工具，本文将介绍大规模集群下 Ansible 的关键配置与优化策略。

## 并发控制

* `--forks`
    Ansible 默认的并发数（forks）是 5，在数十台机器以内尚可接受，但节点数达到成百上千的规模时，执行效率会非常低，建议实时监控控制节点负载，根据硬件能力动态调优以避免过载。

* `serial`
    滚动升级或分批部署时必须启用，建议制定分批数策略，推荐从 5% ~ 10% 集群规模开始，逐步放大，减少因批量操作导致的集群抖动、业务中断风险。

    例如：

    ```yaml
    - hosts: kube_nodes
    serial: 20
    ```

## 超时与重试

* `--timeout`
    Ansible 对 SSH 连接的默认超时时间较短（通常是 10 秒），大规模场景网络抖动可能导致误判超时，推荐调整至 60 ~ 600 秒，视场景灵活配置。

* `retry_stagger`
    控制重试节点间的间隔（秒），防止短时间内过多重试引发控制节点或网络瞬时压力过大。

    例如：

    ```yaml
    retry_stagger: 60
    ```

## 控制节点资源规划

* 控制节点在大规模场景中需要持有连接大量的 SSH 会话，建议开启 SSH 复用，配备 8 core 以上的多核 CPU。

    ```cfg
    [ssh_connection]
    pipelining = True
    ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ConnectionAttempts=100 -o UserKnownHostsFile=/dev/null
    ```

* 控制节点需要充足的内存承担缓存、模板渲染、变量计算等高负载任务，建议配备 16G 以上的内存。
* 控制节点通常会存在大量文件分发、日志持久化等场景，为了提升 I/O 性能，建议使用高速 SSD，特殊场景下考虑使用 NVMe。
* 控制节点建议使用 1Gbps 以上网卡，部署在专用运维子网，避免跨地域调度。

## 动态 inventory

大规模场景中通常会使用动态 inventory，避免重复手动编写 inventory 配置。

注：调用内部 CMDB API 时，应开启认证，避免信息泄露。

<!-- TODO
可参考文章：[如何实现一个自定义 inventory 插件]({{< relref "posts/infra/tools/inventory-plugin.md" >}})
-->

## 日志输出

Ansible 默认会将 playbook 执行日志输出到控制台，建议配置将日志以 JSON 格式写入文件，并结合日志审计平台统一收集，另注意对敏感数据加密或者脱敏处理。

```cfg
log_path=/var/log/ansible.log
stdout_callback = json
```

## 代码和版本管理

* Ansible 上的 playbook、roles、inventory 等均应纳入 Git 代码仓库管理，并结合 CI/CD 流水线进行自动化构建部署包。
* 控制节点应确保使用 Python 虚拟环境或容器镜像进行隔离，`requirements.yml` 下版本锁定防止破坏性变更。
* 不同项目上的 Ansible 的版本应实际项目版本对齐，避免随意升级导致生产事故。

## 安全与权限管理

避免使用 root 账号运行 Ansible。建议使用专用账户（如 ansible 用户）执行自动化任务，利用 ansible_become 机制按需提权，控制节点权限最小化。

定期轮换和管理 SSH 密钥，敏感信息建议使用 Ansible Vault 加密。
