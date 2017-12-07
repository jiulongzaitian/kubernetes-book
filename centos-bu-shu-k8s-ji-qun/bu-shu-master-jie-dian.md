# 部署master节点

```
作者：张杰  j.zhang8@haihangyun.com
```

---

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
wget wget https://dl.k8s.io/v1.8.4/kubernetes-server-linux-amd64.tar.gz
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

`/etc/kubernetes/config`文件的内容为

```
cat > /etc/kubernetes/config << EOF

# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level,  如果想调试可以设置为8 
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver

KUBE_MASTER="--master=http://${IP}:8080"

EOF
```

该配置文件同时被kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy使用。

apiserver配置文件`/etc/kubernetes/apiserver`内容为：

```
export ETCD2=10.72.84.161
export ETCD3=10.72.84.162

export ARGS1="--authorization-mode=RBAC --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --experimental-bootstrap-token-auth"
export ARGS2="--token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem"
export ARGS3="--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem"
export ARGS4="--etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"
export ARGS5="--enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h"
export ARGS6=" "


cat > /etc/kubernetes/apiserver << EOF

###
## kubernetes system config
##
## The following values are used to configure the kube-apiserver
##
#
## The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=${IP} --bind-address=${IP} --insecure-bind-address=${IP}"
#
## The port on the local server to listen on.
#KUBE_API_PORT="--port=8080"
#
## Port minions listen on
#KUBELET_PORT="--kubelet-port=10250"
#
## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://${IP}:2379,https://${ETCD2}:2379,https://${ETCD3}:2379"
#
## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
#
## default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota,PodSecurityPolicy"
#
## Add your own!
KUBE_API_ARGS="$ARGS1 $ARGS2 $ARGS3 $ARGS4 $ARGS5 $ARGS6"
EOF
```

* 注意ETCD2 和 ETCD3 的值， 相信你可以的

* `--authorization-mode=RBAC`指定在安全端口使用 RBAC 授权模式，拒绝未通过授权的请求；

* kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用**非安全端口**和 kube-apiserver通信;kubelet、kube-proxy、kubectl 部署在其它 Node 节点上，如果通过**安全端口**访问 kube-apiserver，则必须先通过 TLS 证书认证，再通过 RBAC 授权；

* kube-proxy、kubectl 通过在使用的证书里指定相关的 User、Group 来达到通过 RBAC 授权的目的；

* 如果使用了 kubelet TLS Boostrap 机制，则不能再指定`--kubelet-certificate-authority`、`--kubelet-client-certificate--kubelet-client-key`选项，否则后续 kube-apiserver 校验 kubelet 证书时出现 ”x509: certificate signed by unknown authority“ 错误；

* `--admission-control`值必须包含`ServiceAccount`；

* `--bind-address`不能为`127.0.0.1`；

* `runtime-config`配置为`rbac.authorization.k8s.io/v1beta1`，表示运行时的apiVersion；

* `--service-cluster-ip-range`指定 Service Cluster IP 地址段，该地址段不能路由可达；

* 缺省情况下 kubernetes 对象保存在 etcd`/registry`路径下，可以通过`--etcd-prefix`参数进行调整；

* 生成配置文件后建议检查配置 文件

* **启动kube-apiserver**

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver -l
```

**创建 kube-controller-manager的serivce配置文件**

文件路径`/usr/lib/systemd/system/kube-controller-manager.service`

```
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/local/bin/kube-controller-manager \\
        \$KUBE_LOGTOSTDERR \\
        \$KUBE_LOG_LEVEL \\
        \$KUBE_MASTER \\
        \$KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

配置文件`/etc/kubernetes/controller-manager`

```
cat > /etc/kubernetes/controller-manager << EOF
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true"

EOF
```

* `--service-cluster-ip-range`参数指定 Cluster 中 Service 的CIDR范围，该网络在各 Node 间必须路由不可达，必须和 kube-apiserver 中的参数一致；

* `--cluster-signing-*`指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥；

* `--root-ca-file`用来对 kube-apiserver 证书进行校验，**指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件**；

* `--address`值必须为`127.0.0.1`，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器，否则：

```
kubectl get componentstatuses

#NAME                 STATUS      MESSAGE                                                                                        ERROR
#controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: getsockopt: connection refused   
#scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: getsockopt: connection refused   
#etcd-0               Healthy     {"health": "true"}                                                                             
#etcd-1               Healthy     {"health": "true"}                                                                             
#etcd-2               Healthy     {"health": "true"}
```

### 启动 kube-controller-manager {#启动-kube-controller-manager}

```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
```

```
kubectl get componentstatuses 

#NAME                 STATUS      MESSAGE                                                                                        ERROR
#scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: getsockopt: connection refused   
#controller-manager   Healthy     ok                                                                                             
#etcd-0               Healthy     {"health": "true"}                                                                             
#etcd-1               Healthy     {"health": "true"}                                                                             
#etcd-2               Healthy     {"health": "true"}
```

## 配置和启动 kube-scheduler {#配置和启动-kube-scheduler}

**创建 kube-scheduler的serivce配置文件**

文件路径`/usr/lib/systemd/system/kube-scheduler.service`。

```
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/local/bin/kube-scheduler \\
            \$KUBE_LOGTOSTDERR \\
            \$KUBE_LOG_LEVEL \\
            \$KUBE_MASTER \\
            \$KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

配置文件`/etc/kubernetes/scheduler`

```
cat > /etc/kubernetes/scheduler << EOF
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"
EOF
```

* --address 值必须为127.0.0.1 因为蛋清的kube-apiserver期望scheduler 在同一台机器
* ### 启动 kube-scheduler {#启动-kube-scheduler}

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
```

## 验证 master 节点功能 {#验证-master-节点功能}

```
kubectl get componentstatuses

#NAME                 STATUS    MESSAGE              ERROR
#controller-manager   Healthy   ok                   
#scheduler            Healthy   ok                   
#etcd-0               Healthy   {"health": "true"}   
#etcd-1               Healthy   {"health": "true"}   
#etcd-2               Healthy   {"health": "true"}
```

## 安装和配置 kubelet-bootstrap {#安装和配置-kubelet}

kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper cluster 角色\(role\)， 然后 kubelet 才能有权限创建认证请求\(certificate signing requests\)：

```
cd /etc/kubernetes
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

`--user=kubelet-bootstrap`是在`/etc/kubernetes/token.csv`文件中指定的用户名，同时也写入了`/etc/kubernetes/bootstrap.kubeconfig`文件；

