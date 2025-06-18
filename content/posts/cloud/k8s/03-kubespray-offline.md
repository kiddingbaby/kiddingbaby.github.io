---
title: "K8s: 03-Kubespray 中的离线部署方案"
date: 2025-06-14T10:00:00+08:00
lastmod: 2025-06-14T10:00:00+08:00
categories: ["云平台"]
tags: ["Kubernetes", "Kubespray", "离线部署", "高可用集群", "私有仓库", "Harbor"]
series: ["K8s"]
keywords: ["Kubespray 离线部署", "Kubernetes Harbor 集成", "内网 Kubernetes 部署", "Kubernetes 高可用生产集群"]
ShowToc: true
TocOpen: true
draft: false
---

## Kubespray 离线部署

在 Kubespray 中不仅对离线部署支持非常完善，还提供了兼容性非常强大的脚本，鉴于官方离线脚本在公开资料中的实践较少，本文将详细介绍基于官方脚本的离线部署方案。

核心组件版本：

```yaml
Kubespray: 2.27.0
```

1. 当前的离线部署目前还有一些小问题，我们先手动修复以下 ansible 代码：

   * 获取 image ID 的命令存在兼容性问题（`contrib/offline/manage-offline-container-images.sh`）

        ```diff
        -image_id=$(sudo ${runtime} image inspect ${org_image} | grep "\"Id\":" | awk -F: '{print $3}'| sed s/'\",'//)
        +image_id=$(sudo ${runtime} image inspect --format "{{.Id}}" "${org_image}")
        ```

   * 设置 `download_run_once=True` 执行 playbook 时控制节点缺少镜像和文件缓存目录（`roles/download/tasks/prep_download.yml`）

        ```diff
        -when:
        -    - download_force_cache
        +when: 
        +    - download_force_cache or download_run_once
        ```

1. 在 kubespray 项目根目录下执行命令，生成镜像和二进制文件清单：

    ```bash
    cd contrib/offline
    bash generate_list.sh
    ```

1. 批量下载二进制文件并打包：

    ```bash
    # NO_HTTP_SERVER=1：禁用临时 HTTP 服务，仅下载文件
    NO_HTTP_SERVER=1 bash ./manage-offline-files.sh
    ```

1. 将 offline-files.tar.gz 上传并解压到离线环境，并配置 Nginx 对外提供 HTTP 服务：

    ```nginx
    server {
        listen 443  ssl;
        server_name offline.example.internal;

        include /etc/openresty/conf.d/ssl.conf;

        access_log      /var/log/openresty/offline-access.log;
        error_log       /var/log/openresty/offline-error.log;

        location = /k8s {
            return 301 /k8s/;
        }

        location ^~ /k8s/ {
            alias /opt/offline-files/;
            index index.html;
            autoindex on;
            autoindex_exact_size off;
            autoindex_localtime on;
            try_files $uri $uri/ = 404;
        }
    }
    ```

1. 拉取集群镜像并打包，默认会在脚本所在目录下生成 `container-images.tar.gz` 归档包：

    ```bash
    # IMAGES_FROM_FILE：从生成的镜像列表拉取镜像并打包成镜像归档
    IMAGES_FROM_FILE=temp/images.list bash ./manage-offline-container-images.sh create
    ```

1. 登录私有仓库，脚本支持 nerdctl、podman、docker 多种 runtime，这里通过 podman 登录：

    ```bash
    podman login harbor.example.internal -u 'username' -p 'password'
    ```

1. 注册镜像到私有仓库，建议在 Harbor 中预先创建对应的 Project，用于管理私有镜像，比如 k8s：

    ```bash
    # DESTINATION_REGISTRY：目标私有仓库地址
    # PRIVATE_REGISTRY：当原始镜像使用私有仓库时设置
    DESTINATION_REGISTRY=harbor.example.internal/k8s bash ./manage-offline-container-images.sh register
    ```

    📌注：如果未指定 DESTINATION_REGISTRY，脚本会默认启动一个本地 registry (port 5000)，并使用 `$(hostname):${REGISTRY_PORT}` 作为目的仓库地址

1. 进入私有仓库确认镜像已注册，配置 `inventory/mycluster/group_vars/all/offline.yml` 文件：

    ```bash
    ---
    ## Global Offline settings
    ### Private Container Image Registry
    registry_host: "harbor.example.internal/k8s"
    files_repo: "https://offline.example.internal/k8s"
    ### If using CentOS, RedHat, AlmaLinux or Fedora
    # yum_repo: "http://myinternalyumrepo"
    ### If using Debian
    # debian_repo: "http://myinternaldebianrepo"
    ### If using Ubuntu
    # ubuntu_repo: "http://myinternalubunturepo"

    ## Container Registry overrides
    kube_image_repo: "{{ registry_host }}"
    gcr_image_repo: "{{ registry_host }}"
    github_image_repo: "{{ registry_host }}"
    docker_image_repo: "{{ registry_host }}"
    quay_image_repo: "{{ registry_host }}"

    ## Kubernetes components
    kubeadm_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubeadm"
    kubectl_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubectl"
    kubelet_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubelet"


    ## Two options - Override entire repository or override only a single binary.

    ## [Optional] 1 - Override entire binary repository
    github_url: "{{ files_repo }}/github.com"
    dl_k8s_io_url: "{{ files_repo }}/dl.k8s.io"
    storage_googleapis_url: "{{ files_repo }}/storage.googleapis.com"
    get_helm_url: "{{ files_repo }}/get.helm.sh"
    ...
    ...
    ```

1. 测试离线方式部署集群

    ```bash
    ansible-playbook -i inventory/mycluster/inventory.ini -e "download_run_once=True download_localhost=False" cluster.yml 
    ```
