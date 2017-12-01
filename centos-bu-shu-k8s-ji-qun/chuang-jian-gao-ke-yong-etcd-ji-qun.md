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

[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos


[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \\
        --name=${ETCD_NAME} \\
        --cert-file=${ETCD_CERT_FILE} \\
        --key-file=${ETCD_KEY_FILE} \\
        --peer-cert-file=${ETCD_PEER_CERT_FILE} \\
        --peer-key-file=${ETCD_PEER_KEY_FILE} \\
        --trusted-ca-file=${ETCD_TRUSTED_CA_FILE} \\
        --peer-trusted-ca-file=${ETCD_PEER_TRUSTED_CA_FILE} \\
        --initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \\
        --listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \\
        --listen-client-urls=${ETCD_LISTEN_CLIENT_URLS} \\
        --advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \\
        --initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \\
        --initial-cluster=${ETCD_INITIAL_CLUSTER} \\
        --initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \\
        --data-dir=${ETCD_DATA_DIR}

Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target

EOF
```



