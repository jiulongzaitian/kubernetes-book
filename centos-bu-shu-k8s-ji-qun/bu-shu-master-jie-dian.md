kubernetes master 节点包含的组件：

* kube-apiserver
* kube-scheduler
* kube-controller-manager

目前这三个组件需要部署在同一台机器上。

* `kube-scheduler   kube-controller-manager  kube-apiserver`三者的功能紧密相关；
* 同时只能有一个`kube-scheduler`、`kube-controller-manager`进程处于工作状态，如果运行多个，则需要通过选举产生一个 leader；



**注**：

* 暂时未实现master节点的高可用 （10.72.84.160）



## TLS 证书文件 {#tls-证书文件}

检查 token.csv 文件

```
ls /etc/kubernetes/token.csv 
#/etc/kubernetes/token.csv
```

## 下载最新版本的二进制文件 {#下载最新版本的二进制文件}

从[`CHANGELOG`页面](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md)下载`client`或`server`tarball 文件

`server`的 tarball`kubernetes-server-linux-amd64.tar.gz`已经包含了`client`\(`kubectl`\) 二进制文件，所以不用单独下载`kubernetes-client-linux-amd64.tar.gz`文件；

```
# wget https://dl.k8s.io/v1.6.0/kubernetes-client-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.6.0/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes
tar -xzvf  kubernetes-src.tar.gz
#将二进制文件拷贝到指定路径
cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/
```

## 配置和启动 kube-apiserver {#配置和启动-kube-apiserver}

**创建 kube-apiserver的service配置文件**

serivce配置文件`/usr/lib/systemd/system/kube-apiserver.service`内容：

```
cat >/usr/lib/systemd/system/kube-apiserver.service << EOF

[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/local/bin/kube-apiserver \\
        \$KUBE_LOGTOSTDERR \\
        \$KUBE_LOG_LEVEL \\
        \$KUBE_ETCD_SERVERS \\
        \$KUBE_API_ADDRESS \\
        \$KUBE_API_PORT \\
        \$KUBELET_PORT \\
        \$KUBE_ALLOW_PRIV \\
        \$KUBE_SERVICE_ADDRESSES \\
        \$KUBE_ADMISSION_CONTROL \\
        \$KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF
```



