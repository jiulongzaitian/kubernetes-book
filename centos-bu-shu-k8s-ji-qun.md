# centos 部署k8s 集群

---

本系列文档介绍使用二进制部署`kubernetes`集群的所有步骤，而不是使用`kubeadm`等自动化方式来部署集群，同时开启了集群的TLS安全认证；

在部署的过程中，将详细列出各组件的启动参数，给出配置文件，详解它们的含义和可能遇到的问题。

部署完成后，你将理解系统各组件的交互原理，进而能快速解决实际问题。

所以本文档主要适合于那些有一定 kubernetes 基础，想通过一步步部署的方式来学习和了解系统配置、运行原理的人。

## 提供所有的配置文件 {#提供所有的配置文件}

集群安装时所有组件用到的配置文件，包含在以下目录中：

* **etc**
  ： service的环境变量配置文件
* **manifest**
  ： kubernetes应用的yaml文件
* **systemd**
  ：systemd serivce配置文件

## 集群详情 {#集群详情}

* Kubernetes 1.6.0
* Docker 1.12.5（使用yum安装）
* Etcd 3.1.5
* Flanneld 0.7 vxlan 网络
* TLS 认证通信 \(所有组件，如 etcd、kubernetes master 和 node\)
* RBAC 授权
* kublet TLS BootStrapping
* kubedns、dashboard、heapster\(influxdb、grafana\)、EFK\(elasticsearch、fluentd、kibana\) 集群插件

## 环境说明 {#环境说明}

在下面的步骤中，我们将在三台CentOS系统的物理机上部署具有三个节点的kubernetes1.6.0集群。

角色分配如下：

**Master**：10.72.84.160

**Node**：10.72.84.160 10.72.84.161 10.72.84.166 10.72.84.167

注意：10.72.84.160这台主机master和node复用。所有生成证书、执行kubectl命令的操作都在这台节点上执行。一旦node加入到kubernetes集群之后就不需要再登陆node节点了。

## 安装前的准备 {#安装前的准备}

1. 在node节点上安装docker1.12.5
2. 关闭所有节点的SELinux
3. 准备harbor私有镜像仓库

## 步骤介绍 {#步骤介绍}

* 1 创建 TLS 证书和秘钥
* 2 创建kubeconfig 文件
* 3 创建高可用etcd集群
* 4 安装kubectl命令行工具
* 5 部署master节点
* 6 部署node节点
* 7 安装kubedns插件
* 8 安装dashboard插件
* 9 安装heapster插件

## 提醒 {#提醒}

1. 由于启用了 TLS 双向认证、RBAC 授权等严格的安全机制，建议
   **从头开始部署**
   ，而不要从中间开始，否则可能会认证、授权等失败！
2. 部署过程中需要有很多证书的操作，请大家耐心操作，不明白的操作可以参考本书中的其他章节的解释。
3. 该部署操作仅是搭建成了一个可用 kubernetes 集群，而很多地方还需要进行优化，heapster 插件、EFK 插件不一定会用于真实的生产环境中，但是通过部署这些插件，可以让大家了解到如何部署应用到集群上。

**注：本安装文档参考了**[**opsnull 跟我一步步部署 kubernetes 集群**](https://github.com/opsnull/follow-me-install-kubernetes-cluster/)

