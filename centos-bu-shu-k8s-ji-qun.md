# 部署k8s 集群

```
作者：张杰  j.zhang8@haihangyun.com
```

---

本系列文档介绍使用二进制部署`kubernetes`集群的所有步骤，而不是使用`kubeadm`等自动化方式来部署集群，同时开启了集群的TLS安全认证；

在部署的过程中，将详细列出各组件的启动参数，给出配置文件，详解它们的含义和可能遇到的问题。

部署完成后，你将理解系统各组件的交互原理，进而能快速解决实际问题。

所以本文档主要适合于那些有一定 kubernetes 基础，想通过一步步部署的方式来学习和了解系统配置、运行原理的人。

## 提供所有的配置文件 {#提供所有的配置文件}

集群安装时所有组件用到的配置文件，包含在以下目录中：

* **etc**： service的环境变量配置文件
* **manifest**： kubernetes应用的yaml文件
* **systemd**：systemd serivce配置文件

## 集群详情 {#集群详情}

* Kubernetes 1.6.0
* Docker 1.12.5（使用yum安装）
* Etcd 3.1.5
* Flanneld 0.7 vxlan 网络
* TLS 认证通信 \(所有组件，如 etcd、kubernetes master 和 node\)
* RBAC 授权
* kublet TLS BootStrapping
* kubedns、dashboard、heapster\(influxdb、grafana\)、EFK\(elasticsearch、fluentd、kibana\) 集群插件
* 操作系统 centos 7

## 环境说明 {#环境说明}

在下面的步骤中，我们将在三台CentOS系统的物理机上部署具有三个节点的kubernetes1.6.0集群。

角色分配如下：

**Master**：10.72.84.160

**Node**：10.72.84.160 10.72.84.161 10.72.84.162 10.72.84.166 10.72.84.167

注意：10.72.84.160这台主机master和node复用。所有生成证书、执行kubectl命令的操作都在这台节点上执行。一旦node加入到kubernetes集群之后就不需要再登陆node节点了。

## 安装前的准备 {#安装前的准备}

1. 在node节点上安装docker1.12.5
2. 关闭所有节点的SELinux （重启）
3. 准备harbor私有镜像仓库
4. 所有主机执行下面的命令

```
yum install docker telnet wget nfs-utils net-tools -y

modprobe br_netfilter && sysctl net.bridge.bridge-nf-call-iptables=1
modprobe br_netfilter && sysctl net.bridge.bridge-nf-call-iptables=1

systemctl disable firewalld && systemctl stop firewalld
```

开机加载

```
vim /etc/modules-load.d/modules.conf
```

添加下面一行

```
br_netfilter
```

vim  /usr/lib/sysctl.d/00-system.conf

```
# 将 net.bridge.bridge-nf-call-iptables的值改为1，
# 之后reboot
# 不同环境net.bridge.bridge-nf-call-iptables 设置位置不一样，
# 查看 /etc/sysctl.conf  的说明
```

1. 每个主机上设置IP的环境变量，

以10.72.84.160 这台主机为例

```
IP=$(ifconfig eth0 |grep inet |grep netmask |grep broadcast |awk -F " " '{printf $2 }')
echo "export IP=${IP}" >> ~/.bashrc
source ~/.bashrc

echo "验证IP 结果 IP=$IP"
```

##  {#步骤介绍}

## 步骤介绍 {#步骤介绍}

* [创建 TLS 证书和秘钥](/centos-bu-shu-k8s-ji-qun/chuang-jian-tls-zheng-shu-he-mi-yao.md)
* [安装kubectl命令行工具](https://www.gitbook.com/book/jiulongzaitian/kubernetes/edit#)

* [创建kubeconfig 文件](/centos-bu-shu-k8s-ji-qun/chuang-jian-kubeconfig-wen-jian.md)

* [创建高可用etcd集群](/centos-bu-shu-k8s-ji-qun/chuang-jian-gao-ke-yong-etcd-ji-qun.md)

* [部署master节点](/centos-bu-shu-k8s-ji-qun/bu-shu-master-jie-dian.md)

* [部署node节点](/centos-bu-shu-k8s-ji-qun/bu-shu-node-jie-dian.md)

* [ 安装kubedns插件](/centos-bu-shu-k8s-ji-qun/an-zhuang-kubedns-cha-jian.md)

* [安装dashboard插件](/centos-bu-shu-k8s-ji-qun/an-zhuang-dashboard-cha-jian.md)

* [安装heapster插件](/centos-bu-shu-k8s-ji-qun/an-zhuang-heapster-cha-jian.md)

## 提醒 {#提醒}

1. 由于启用了 TLS 双向认证、RBAC 授权等严格的安全机制，建议
   **从头开始部署**
   ，而不要从中间开始，否则可能会认证、授权等失败！
2. 部署过程中需要有很多证书的操作，请大家耐心操作，不明白的操作可以参考本书中的其他章节的解释。
3. 该部署操作仅是搭建成了一个可用 kubernetes 集群，而很多地方还需要进行优化，heapster 插件、EFK 插件不一定会用于真实的生产环境中，但是通过部署这些插件，可以让大家了解到如何部署应用到集群上。

**注：本安装文档参考了**[**opsnull 跟我一步步部署 kubernetes 集群**](https://github.com/opsnull/follow-me-install-kubernetes-cluster/)



[https://github.com/kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

