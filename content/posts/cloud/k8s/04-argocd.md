---
title: "K8s: 04-利用 ArgoCD 部署系统级组件"
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

## ArgoCD 配置

1. 配置 HTTP Ingress

```bash
cat <<EOF > argocd-server-http-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    cert-manager.io/cluster-issuer: ca-issuer
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
    host: argocd.example.internal
  tls:
  - hosts:
    - argocd.example.internal
    secretName: argocd-server-http-cert-tls
EOF

kubectl apply -f argocd-server-http-ingress.yaml
```

1. 配置 GRPC Ingress

```bash
cat <<EOF > argocd-server-grpc-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-grpc-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
    host: argocd-grpc.example.internal
  tls:
  - hosts:
    - argocd-grpc.example.internal
    secretName: argocd-server-grpc-cert-tls
EOF

kubectl apply -f argocd-server-grpc-ingress.yaml
```

1. 配置访问凭据

ArgoCD 需要凭据来访问 Git 仓库（存放应用声明）和 Kubernetes API（部署应用）。

Git 凭据：

对于私有 Git 仓库，SSH Key 是最安全和推荐的方式。
生成 SSH Key 对。
将公钥添加到您的 Git 仓库。
将私钥作为 Kubernetes Secret 存储在 argocd 命名空间：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```
在 ArgoCD 中注册 Git 仓库，引用此 Secret。

## 配置初始 RBAC

## 配置访问 OpenBao（Vault 分支）的 secret 令牌

## 加载 Ansible Vault 中加密的机密变量到 Kubernetes Secret

## 部署 bootstrap.yaml （App of Apps）

## 所有资源由 Ansible 最后执行

## ArgoCD 同步自身

## 部署系统级组件
