---
title: "K8s: 04-标签治理方案"
date: 2025-06-14T10:00:00+08:00
lastmod: 2025-06-14T10:00:00+08:00
categories: ["云平台"]
tags: ["Kubernetes", "Kubespray", "离线部署", "高可用集群", "私有仓库", "Harbor"]
series: ["K8s"]
keywords: ["Kubespray 离线部署", "Kubernetes Harbor 集成", "内网 Kubernetes 部署", "Kubernetes 高可用生产集群"]
ShowToc: true
TocOpen: true
draft: true
---

标签（Labels）作为 Kubernetes 对象元数据的重要组成部分，缺乏统一的标签规范往往会导致资源混乱、难以追溯，并阻碍自动化能力的进一步提升，本人初步构建了一套易于理解与落地的 Kubernetes 标签治理方案，供大家分享。

## 设计目标

| 目标   | 预期效果                                                             | 验证方式                               |
| ------ | -------------------------------------------------------------------- | -------------------------------------- |
| 一致性 | 统一标签结构，支持各种生态工具无歧义识别。                           | kubectl、prometheus、kubecost 标签查询 |
| 可追溯 | 标签覆盖业务线、环境、团队、成本中心，支持审计、成本核算和策略追踪。 | GitOps diff、ArgoCD app diff           |
| 易扩展 | 支持多集群、多环境、多团队场景，方便治理和未来扩展。                 | 跨集群标签查询、Pod affinity 验证      |

## 集群命名规范

集群命名旨在唯一标识 Kubernetes 的物理实例。一个逻辑环境（如 `dev`、`prod`）可能由多个物理集群实例组成。

| 集群名示例                  | 说明                                |
| --------------------------- | ----------------------------------- |
| `local`                     | 本地（开发者桌面）实验环境          |
| `dev-cn-hangzhou-01`        | 国内杭州开发环境集群实例 1          |
| `test-cn-hangzhou-01`       | 国内杭州测试环境集群实例 1          |
| `staging-cn-hangzhou-01`    | 国内杭州预生产环境集群实例 1        |
| `prod-aws-us-east-01`       | AWS 美东区域生产环境集群实例 1 - HA |
| `prod-aws-us-east-02`       | AWS 美东区域生产环境集群实例 2 - HA |
| `prod-aws-us-central-dr-01` | AWS 美中区域生产环境灾备集群实例 1  |

## Namespace 命名规范

| Namespace       | 说明           | 示例    |
| --------------- | -------------- | ------- |
| `<productline>` | 业务产品线名称 | payment |

📌注： Namespace 的标签不会继承给 Workload 资源，业务层资源应独立打标签。

### Namespace 推荐标签

| 标签键                         | 说明                   | 推荐示例                  | 备注                                               |
| ------------------------------ | ---------------------- | ------------------------- | -------------------------------------------------- |
| `example.internal/owner-team`  | 整体管理与治理归属团队 | `core-product-governance` | 产品线所属业务部门，便于成本核算和整体业务责任划分 |
| `example.internal/cost-center` | 所属成本核算中心编码   | `cc-12345`                | 可选项，若企业有按 Namespace 归属成本的需求        |

## 标签命名规范

### 标签键

遵循 Kubernetes 标签键命名约定：

- 必须使用 `<prefix>/<name>` 格式，前缀建议为组织域名（如 `example.internal`）
- `<name>` 仅允许小写字母、数字、连字符 `-`，长度不超过 63 个字符
- 总长度不超过 253 个字符
- 应全部小写，禁止大小写混用

### 标签值

遵循 Kubernetes 标签值命名约定：

- 长度不超过 63 个字符
- 仅允许小写字母、数字、连字符 `-` 和点号 `.`
- 必须以字母或数字开头和结尾

| 标签键                         | 必选 | 说明                        | 推荐示例                                | 备注                                         |
| ------------------------------ | ---- | --------------------------- | --------------------------------------- | -------------------------------------------- |
| `app.kubernetes.io/name`       | Y    | 微服务唯一标识              | `payment-api`                           |                                              |
| `app.kubernetes.io/version`    | Y    | 应用正式版本号，SemVer 格式 | `1.0.0`                                 | 用于灰度、回滚。                             |
| `app.kubernetes.io/instance`   | N    | 应用的特定部署实例名        | `payment-api-canary`、`payment-api-v2`  | 推荐场景：蓝绿/金丝雀发布、多版本并行等      |
| `app.kubernetes.io/part-of`    | Y    | 微服务所属逻辑系统          | `transaction-core`                      | 参考产品-系统映射表                          |
| `app.kubernetes.io/component`  | Y    | 微服务所属模块/功能职责     | `api`、`worker`、`consumer`、`database` | 无明确业务模块划分时，可参考通用组件名列表。 |
| `app.kubernetes.io/managed-by` | Y    | 管理工具名称                | `helm`、`argocd`、`kustomize`           | 自动注入                                     |
| `example.internal/environment` | Y    | 业务环境逻辑标识            | `dev`、`test`、`staging`、`prod`        | 参考环境名列表                               |
| `example.internal/cluster`     | Y    | 集群唯一标识                | `dev-01`、`dev-02`、`prod-01`           | 参考集群名列表                               |
| `example.internal/team`        | Y    | 所属团队（小组）            | `payment-backend`                       | 参考团队名称列表                             |
| `example.internal/cost-center` | N    | 所属成本核算中心编码        | `cc-12345`                              | 依赖成本管理系统，编码需保持一致             |

说明事项：

- 建议所有资源（Pod、Deployment、Service、ConfigMap 等）必须完整打上必选标签，禁止无标签资源上线。
- GitOps 流程中，所有标签应在代码仓库中定义并受版本控制。
- 版本号使用纯 SemVer 格式，方便灰度发布和版本回溯。
- 管理工具标签便于自动化识别和管理。
- 后续应使用 OPA 等工具强制执行标签规范，确保规范落地。
- 建议在 CI/CD 管道中引入标签预校验，以在部署前即时发现并纠正不符合规范的标签。
