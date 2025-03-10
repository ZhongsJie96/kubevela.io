---
title: 招商银行 KubeVela 离线部署实践
author: 马祥博
author_title: （云平台开发团队）
author_url: http://www.cmbchina.com/
author_image_url: /img/china-merchants-bank.jpg
tags: [ KubeVela ]
description: ""
image: https://raw.githubusercontent.com/kubevela/kubevela.io/main/docs/resources/KubeVela-03.png
hide_table_of_contents: false
---

招商银行云平台开发团队自2021年开始接触 KubeVela，并探索 KubeVela 在招商银行云平台的落地实践，借此提升云原生应用交付与管理能力。同时因为金融保险行业的特殊性，网络安全管控措施相对严格，行内网络无法直接拉取 Docker Hub 镜像，同时行内暂时没有可用的 Helm 镜像源。因此，要想实现 KubeVela 在行内私有环境的落地，必须进行完全的离线部署。

本文将以 KubeVela v1.2.5 版本为例，介绍招商银行 KubeVela 的离线部署实践，来帮助其他用户在离线环境中更便捷的完成 KubeVela 的部署。

## KubeVela 离线部署方案

我们将 KubeVela 的离线部署主要分为三部分，分别是 Vela Cli、Vela Core 以及 Addon 的离线部署，每一部分主要涉及到相关 docker 镜像的加载及 Helm 的 repackage，通过该离线部署方案，能够大大加快 KubeVela 在离线环境的部署。

在离线部署前请确保 Kubernetes 集群版本 `>= v1.19 && < v1.22`，KubeVela 控制平面依赖 Kubernetes，可以放置在任何托管 Kubernetes 作为底座的产品或自建 Kubernetes 集群中。同时你也可以使用 kind 或 minikube 在本地部署、测试 KubeVela。

### Vela Cli 离线部署

- 首先，需要通过 KubeVela 的 [发布日志](https://github.com/kubevela/kubevela/releases) 下载你所需版本的 `vela` 二进制文件
- 解压二进制文件，并且在 `$PATH` 中配置相应的环境变量 
   - 解压二进制文件 
      - `tar -zxvf vela-v1.2.5-linux-amd64.tar.gz`
      - `mv ./linux-amd64/vela /usr/local/bin/vela`
   - 设置环境变量 
      - `vi /etc/profile`
      - `export PATH="$PATH:/usr/local/bin"`
      - `source /etc/profile`
   - 通过 `vela version` 验证 Vela Cli 的安装，并检查输出
```shell
CLI Version: v1.2.5
Core Version:
GitRevision: git-ef80b66
GolangVersion: go1.17.7
```

- 至此，Vela Cli 已经离线部署完成！

### Vela Core 离线部署

- 离线部署 Vela Core 之前，首先需要在离线环境中 [安装 Helm](https://helm.sh/docs/intro/install/) ， 并且 Helm 的版本需要满足`v3.2.0+`
- 准备 docker 镜像， Vela Core 的部署主要涉及5个镜像，你需要首先访问互联网从 Docker Hub 下载相应镜像，之后再 load 到离线环境 
   - 从Docker Hub拉取镜像 
      - `docker pull oamdev/vela-core:v1.2.5`
      - `docker pull oamdev/cluster-gateway:v1.1.7`
      - `docker pull oamdev/kube-webhook-certgen:v2.3`
      - `docker pull oamdev/alpine-k8s:1.18.2`
      - `docker pull oamdev/hello-world:v1`
   - 将镜像保存到本地磁盘 
      - `docker save -o vela-core.tar oamdev/vela-core:v1.2.5`
      - `docker save -o cluster-gateway.tar oamdev/cluster-gateway:v1.1.7`
      - `docker save -o kube-webhook-certgen.tar oamdev/kube-webhook-certgen:v2.3`
      - `docker save -o alpine-k8s.tar oamdev/alpine-k8s:1.18.2`
      - `docker save -o hello-world.tar oamdev/hello-world:v1`
   - 在私有环境中重新加载镜像 
      - `docker load vela-core.tar`
      - `docker load cluster-gateway.tar`
      - `docker load kube-webhook-certgen.tar`
      - `docker load alpine-k8s.tar`
      - `docker load hello-world.tar`
- 下载 [KubeVela 源码](https://github.com/kubevela/kubevela/releases) ，拷贝到离线环境中，并使用 Helm 重新打包 
   - 将 KubeVela 源码重新打 chart 包，并离线安装 chart 包到控制集群 
      - `helm package kubevela/charts/vela-core --destination kubevela/charts`
      - `helm install --create-namespace -n vela-system kubevela kubevela/charts/vela-core-0.1.0.tgz --wait`
   - 检查输出
```shell
KubeVela control plane has been successfully set up on your cluster.
```

- 至此，Vela Core 已经离线部署完成！

### Addon 离线部署

- 首先下载 [Catalog 源码](https://github.com/kubevela/catalog) 并拷贝到私有环境中
- 这里将以 VelaUX 为例介绍 Addon 的离线部署，首先准备 docker 镜像，VelaUX 主要涉及2个镜像，需要首先访问互联网从 Docker Hub 下载相应镜像，之后再 load 到离线环境 
   - 从 Docker Hub 拉取镜像 
      - `docker pull oamdev/vela-apiserver:v1.2.5`
      - `docker pull oamdev/velaux:v1.2.5`
   - 将镜像保存到本地磁盘 
      - `docker save -o vela-apiserver.tar oamdev/vela-apiserver:v1.2.5`
      - `docker save -o velaux.tar oamdev/velaux:v1.2.5`
   - 在私有环境中重新加载镜像 
      - `docker load vela-apiserver.tar`
      - `docker load velaux.tar`
- 安装 VelaUX 
   - 通过 Vela Cli 安装VelaUX 
      - `vela addon enable catalog-master/addons/velaux`
   - 检查输出
```shell
  Addon: velaux enabled Successfully.
```

   - 若有集群中安装了 route Controller 或 Nginx Ingress Controller，且有可用域名，你可以部署外部路由访问 VelaUX，这里以 openshift route 为例，也可以选择 ingress
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
name: velaux-route
namespace: vela-system
spec:
host: velaux.xxx.xxx.cn
port:
  targetPort: 80
to:
  kind: Service
  name: velaux
  weight: 100
wildcardPolicy: None
```

   - 检查安装
```shell
curl -I -m 10 -o /dev/null -s -w %{http_code} http://velaux.xxx.xxx.cn/applications
```

- 至此，VelaUX 已经离线部署完成！ 同时，对于其他类型 Addon 的离线部署，只需要去 Catalog 源码的对应目录确定所需镜像，并重复以上操作即可完成相应 Addon 的离线部署

## 总结

在离线部署的过程中，我们也尝试将 Vela Core 和 Addon 在互联网环境中部署后产生的资源实例保存为 yaml 文件，并在私有环境中进行重新部署，从而完成离线部署，但由于涉及的资源实例较多以及服务授权问题，导致该种方式较为繁琐。

通过 KubeVela 离线部署实践，可以帮助你更便捷的在离线环境中搭建一整套的 KubeVela，探索 KubeVela 的落地实践。针对离线部署这个共性的问题，我们也看到 KubeVela 社区即将推出全新的 [velad](https://github.com/oam-dev/velad)，一个完全离线、数据高可用的安装工具。Velad 可以帮助自动化完成准备集群、下载打包镜像并安装到离线环境等一系列步骤。它支持了：在 Linux 机器（例如阿里云 ECS）本地启动集群、安装 vela-core；在快速启动一个 KubeVela 控制平面的同时，不必担心控制平面的数据随着机器关机等情况而丢失；velad 可以将控制平面全部数据存储到一个传统数据库（例如 RDS 或另一个 ECS 上部署的 MySQL）。

近期的版本中，招行将加大在 KubeVela 开源社区的投入，积极共建，在企业级集成能力、多集群能力增强、离线部署和应用级统一可观测等诸多领域，贡献来自于金融行业的特定用户场景和业务需求，推动云原生生态实现更易用更高效的应用管理平台向前发展，也欢迎更多的社区成员一起加入进来。