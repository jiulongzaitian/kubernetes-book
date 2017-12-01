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
cp etcd-v3.1.11-linux-amd64/etcd* /usr/local/bin
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
        --name=\${ETCD_NAME} \\
        --cert-file=\${ETCD_CERT_FILE} \\
        --key-file=\${ETCD_KEY_FILE} \\
        --peer-cert-file=\${ETCD_PEER_CERT_FILE} \\
        --peer-key-file=\${ETCD_PEER_KEY_FILE} \\
        --trusted-ca-file=\${ETCD_TRUSTED_CA_FILE} \\
        --peer-trusted-ca-file=\${ETCD_PEER_TRUSTED_CA_FILE} \\
        --initial-advertise-peer-urls=\${ETCD_INITIAL_ADVERTISE_PEER_URLS} \\
        --listen-peer-urls=\${ETCD_LISTEN_PEER_URLS} \\
        --listen-client-urls=\${ETCD_LISTEN_CLIENT_URLS} \\
        --advertise-client-urls=\${ETCD_ADVERTISE_CLIENT_URLS} \\
        --initial-cluster-token=\${ETCD_INITIAL_CLUSTER_TOKEN} \\
        --initial-cluster=\${ETCD_INITIAL_CLUSTER} \\
        --initial-cluster-state=\${ETCD_INITIAL_CLUSTER_STATE} \\
        --data-dir=\${ETCD_DATA_DIR}

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF
```

指定

* `etcd`的工作目录为`/var/lib/etcd`，数据目录为`/var/lib/etcd`，需在启动服务前创建这两个目录；
* 为了保证通信安全，需要指定 etcd 的公私钥\(cert-file和key-file\)、Peers 通信的公私钥和 CA 证书\(peer-cert-file、peer-key-file、peer-trusted-ca-file\)、客户端的CA证书（trusted-ca-file）；
* 创建`kubernetes.pem`证书时使用的`kubernetes-csr.json`文件的`hosts`字段**包含所有 etcd 节点的IP**，否则证书校验会出错；
* `--initial-cluster-state`值为`new`时，`--name`的参数值必须位于`--initial-cluster`列表中；

环境变量配置文件`/etc/etcd/etcd.conf`

```
cat > /etc/etcd/etcd.conf << EOF
ETCD_NAME="160"
ETCD_CERT_FILE="/etc/kubernetes/ssl/kubernetes.pem"
ETCD_KEY_FILE="/etc/kubernetes/ssl/kubernetes-key.pem"
ETCD_PEER_CERT_FILE="/etc/kubernetes/ssl/kubernetes.pem"
ETCD_PEER_KEY_FILE="/etc/kubernetes/ssl/kubernetes-key.pem"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem" 
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://${IP}:2380"
ETCD_LISTEN_PEER_URLS="https://${IP}:2380"
ETCD_LISTEN_CLIENT_URLS="https://${IP}:2379,https://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://${IP}:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster1"
ETCD_INITIAL_CLUSTER=160=https://${IP}:2380,161=https://10.72.84.161:2380,162=https://10.72.84.162:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"

EOF
```

这是10.72.84.160 节点的配置，如果在其他节点上配置，请注意以下变量：

```
ETCD_NAME 
ETCD_INITIAL_CLUSTER
```

注意ETCD\_NAME的值 在 ETCD\_INITIAL\_CLUSTER 的对应关系，相信你的智商 ，你可以的

## 启动 etcd 服务 {#启动-etcd-服务}

```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd -l
```

另外2个etcd节点重复上面的步骤，直到所有机器的 etcd 服务都已启动。

## 验证服务 {#验证服务}

在任一 kubernetes master 机器上执行如下命令：

因为etcd 加入证书，etcdctl 访问时候需要带证书，因为每次带证书比较麻烦，可以设置alase

```
echo "alias etcdctl2='etcdctl --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/kubernetes/ssl/kubernetes.pem --key-file=/etc/kubernetes/ssl/kubernetes-key.pem --endpoints=https://${IP}:2379 '" >> ~/.bashrc
echo "alias etcdctl3='ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/ssl/ca.pem   --cert=/etc/kubernetes/ssl/kubernetes.pem   --key=/etc/kubernetes/ssl/kubernetes-key.pem  --endpoints=https://${IP}:2379 '" >> ~/.bashrc
source ~/.bashrc
```

ETCD V2:

```
etcdctl2 cluster-health

#2017-12-01 10:27:45.859693 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
#2017-12-01 10:27:45.860945 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
#member c87c90d109e3e1f7 is healthy: got healthy result from https://10.72.84.160:2379
#cluster is healthy
```

ETCD V3:

```
etcdctl3 endpoint health

#2017-12-01 10:29:06.210264 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
#https://10.72.84.160:2379 is healthy: successfully committed proposal: took = 1.837169ms
```

访问kubernetes 数据：

```
etcdctl3  get /registry/namespaces/default -w=json|python -m json.tool
```

使用`--prefix`可以看到所有的子目录，如查看集群中的namespace：

```
etcdctl3 get /registry/namespaces --prefix -w=json|python -m json.tool
```

key的值是经过base64编码，需要解码后才能看到实际值，如

```
echo L3JlZ2lzdHJ5L25hbWVzcGFjZXMvYXV0b21vZGVs|base64 -d

#/registry/namespaces/automodel
```



