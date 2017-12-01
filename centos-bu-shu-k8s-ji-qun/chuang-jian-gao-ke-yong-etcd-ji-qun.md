# 创建高可用 etcd 集群 {#创建高可用-etcd-集群}

kuberntes 系统使用 etcd 存储所有数据，本文档介绍部署一个三节点高可用 etcd 集群的步骤，ip：

10.72.84.160  10.72.84.161  10.72.84.162

## TLS 认证文件 {#tls-认证文件}

需要为 etcd 集群创建加密通信的 TLS 证书，这里复用以前创建的 kubernetes 证书

```
cp ca.pem kubernetes-key.pem kubernetes.pem /etc/kubernetes/ssl
```

* kubernetes 证书的hosts字段列表必须包含etcd 集群的三个集群的IP 否则后续证书会校验失败

## 下载二进制文件 {#下载二进制文件}

到`https://github.com/coreos/etcd/releases`页面下载最新版本的二进制文件

```
wget https://github.com/coreos/etcd/releases/download/v3.1.11/etcd-v3.1.11-linux-amd64.tar.gz
tar -xvf etcd-v3.1.11-linux-amd64.tar.gz
mv etcd-v3.1.11-linux-amd64/etcd* /usr/local/bin
```

## 创建 etcd 的 systemd unit 文件 {#创建-etcd-的-systemd-unit-文件}

注意替换IP地址为你自己的etcd集群的主机IP

```
cat > /etc/systemd/system/etcd.service << EOF


EOF
```



