# 部署node节点 {#部署node节点}

```
作者：张杰  j.zhang8@haihangyun.com
```

---

kubernetes node 节点包含如下组件：

* Flanneld：需要在service配置文件中增加TLS配置。
* Docker1.12.5：docker的安装很简单，这里也不说了。
* kubelet
* kube-proxy

下面着重讲`kubelet`和`kube-proxy`的安装，同时还要将之前安装的flannel集成TLS验证。

**注意**：每台 node 上都需要安装 flannel，master 节点上可以不必安装。

以 node : 10.72.84.161  为例

## 目录和文件 {#目录和文件}

我们再检查一下三个节点上，经过前几步操作我们已经创建了如下的证书和配置文件。

```
ls /etc/kubernetes/ssl
#admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem


ls /etc/kubernetes/
#apiserver  bootstrap.kubeconfig  config  controller-manager  kubelet  kube-proxy.kubeconfig  proxy  scheduler  ssl  token.csv
```

## 配置Flanneld {#配置flanneld}

现在需要在serivce配置文件中增加TLS配置。

直接使用yum安装flanneld即可。

```
yum install -y flannel
```

service配置文件 `/usr/lib/systemd/system/flanneld.service`

```
cat > /usr/lib/systemd/system/flanneld.service << EOF

[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=/usr/bin/flanneld-start \\
  -etcd-endpoints=\${ETCD_ENDPOINTS} \\
  -etcd-prefix=\${ETCD_PREFIX} \\
  \$FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```

`/etc/sysconfig/flanneld`配置文件。

```
export ENDPOINT1=10.72.84.160
export ENDPOINT2=10.72.84.161
export ENDPOINT3=10.72.84.162
export ETCD_PREFIX=/kube-centos/network

cat > /etc/sysconfig/flanneld <<EOF
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
ETCD_ENDPOINTS="https://${ENDPOINT1}:2379,https://${ENDPOINT2}:2379,https://${ENDPOINT3}:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
ETCD_PREFIX="${ETCD_PREFIX}"

# Any additional options that you want to pass
FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"
EOF
```

**注意**：ENDPOINT1 ENDPOINT2 ENDPOINT3这三个环境变量，相信你可以的

在FLANNEL\_OPTIONS中增加TLS的配置。

**在etcd中创建网络配置**

执行下面的命令为docker分配IP地址段。 用etcd V2

**注意： 只需要执行一次**

```
etcdctl2 mkdir ${ETCD_PREFIX}
etcdctl2 mk ${ETCD_PREFIX}/config '{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'

#2017-12-01 13:08:27.672156 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
#{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}
```

如果你要使用`host-gw`模式，可以直接将vxlan改成`host-gw`即可。

**注**：参考[网络和集群性能测试](https://jimmysong.io/kubernetes-handbook/practice/network-and-cluster-perfermance-test.html)那节，最终我们使用的`host-gw`模式。

**配置Docker**

```
yum install docker -y

#1.12版本

systemctl restart docker
```

Flannel的[文档](https://github.com/coreos/flannel/blob/master/Documentation/running.md)中有写**Docker Integration**：

> Docker daemon accepts`--bip`argument to configure the subnet of the docker0 bridge. It also accepts`--mtu`to set the MTU for docker0 and veth devices that it will be creating. Since flannel writes out the acquired subnet and MTU values into a file, the script starting Docker can source in the values and pass them to Docker daemon:

```
source /run/flannel/subnet.env
docker daemon --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU} &
```

Systemd users can use`EnvironmentFile`directive in the .service file to pull in`/run/flannel/subnet.env`

yum 安装的flaaneld ，会有一个 **mk-docker-opts.sh 文件，具体路径是：/usr/libexec/flannel/mk-docker-opts.sh**

这个文件是用来`Generate Docker daemon options based on flannel env file`。

**创建  /run/flannel/subnet.env 文件：**

```
cat >  /run/flannel/subnet.env << EOF
FLANNEL_NETWORK=172.30.0.0/16
FLANNEL_SUBNET=172.30.46.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
EOF
```

执行 /usr/libexec/flannel/mk-docker-opts.sh -i  命令

```
/usr/libexec/flannel/mk-docker-opts.sh -i 

# 执行完后会生成 /run/docker_opts.env 文件

cat /run/docker_opts.env

#DOCKER_OPT_BIP="--bip=172.30.46.1/24"
#DOCKER_OPT_IPMASQ="--ip-masq=true"
#DOCKER_OPT_MTU="--mtu=1450"
```

**设置docker0网桥的IP地址**

```
source /run/flannel/subnet.env
ifconfig docker0 $FLANNEL_SUBNET

ifconfig docker0

#docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
#        inet 172.30.46.1  netmask 255.255.255.0  broadcast 172.30.46.255
#        ether 02:42:b7:22:d5:90  txqueuelen 0  (Ethernet)
#        RX packets 0  bytes 0 (0.0 B)
#        RX errors 0  dropped 0  overruns 0  frame 0
#        TX packets 0  bytes 0 (0.0 B)
#        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

**重启flnanneld**

```
systemctl daemon-reload
systemctl enable flanneld
systemctl restart flanneld
systemctl status flanneld
```

同时在 docker 的配置文件 docker.service 中增加环境变量配置：

```
vim /usr/lib/systemd/system/docker.service

EnvironmentFile=-/run/flannel/docker
EnvironmentFile=-/run/docker_opts.env
EnvironmentFile=-/run/flannel/subnet.env
```

**启动docker**

```
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
```

查询etcd内容

```
etcdctl2 ls /kube-centos/network/subnets/

#2017-12-01 17:20:52.017335 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
#/kube-centos/network/subnets/172.30.20.0-24

etcdctl2 get /kube-centos/network/config
#2017-12-01 17:21:37.503071 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
#{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"host-gw"}}

# 若有数据可以查看下面的命令
etcdctl2 get /kube-centos/network/subnets/172.30.14.0-24
```

### 下载最新的 kubelet 和 kube-proxy 二进制文件 {#下载最新的-kubelet-和-kube-proxy-二进制文件}

```
wget https://dl.k8s.io/v1.8.4/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes
tar -xzvf  kubernetes-src.tar.gz
cp -r ./server/bin/{kube-proxy,kubelet} /usr/local/bin/
```

### 创建 kubelet 的service配置文件 {#创建-kubelet-的service配置文件}

文件位置`/usr/lib/systemd/system/kubelet.service`

```
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/bin/kubelet \\
            \$KUBE_LOGTOSTDERR \\
            \$KUBE_LOG_LEVEL \\
            \$KUBELET_API_SERVER \\
            \$KUBELET_ADDRESS \\
            \$KUBELET_PORT \\
            \$KUBELET_HOSTNAME \\
            \$KUBE_ALLOW_PRIV \\
            \$KUBELET_POD_INFRA_CONTAINER \\
            \$KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target


EOF
```

kubelet的配置文件`/etc/kubernetes/kubelet`。其中的IP地址更改为你的每台node节点的IP地址。

注意：`/var/lib/kubelet`需要手动创建。

```
mkdir -p /var/lib/kubelet


cat >/etc/kubernetes/kubelet  << EOF
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=${IP}"
#
## The port for the info server to serve on
#KUBELET_PORT="--port=10250"
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=${IP}"
#
## location of the api-server
KUBELET_API_SERVER="--api-servers=http://${IP}:8080"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=pod-infrastructure:rhel7"
#
## Add your own!
KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=10.254.0.2 --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --require-kubeconfig --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false"
EOF
```

**注意 KUBELET\_POD\_INFRA\_CONTAINER 变量 这是是infra 镜像的地址**

* `--address`不能设置为`127.0.0.1`，否则后续 Pods 访问 kubelet 的 API 接口时会失败，因为 Pods 访问的

`127.0.0.1`指向自己而不是 kubelet；

* 如果设置了`--hostname-override`选项，则`kube-proxy`也需要设置该选项，否则会出现找不到 Node 的情况；
* `"--cgroup-driver`配置成`systemd`，不要使用`cgroup`，否则在 CentOS 系统中 kubelet 将启动失败（保持docker和kubelet中的cgroup driver配置一致即可，不一定非使用`systemd`）。
* `--experimental-bootstrap-kubeconfig`指向 bootstrap kubeconfig 文件，kubelet 使用该文件中的用户名和 token 向 kube-apiserver 发送 TLS Bootstrapping 请求；
* 管理员通过了 CSR 请求后，kubelet 自动在`--cert-dir`目录创建证书和私钥文件\(`kubelet-client.crt`和`kubelet-client.key`\)，然后写入`--kubeconfig`文件；
* 建议在`--kubeconfig`配置文件中指定`kube-apiserver`地址，如果未指定`--api-servers`选项，则必须指定`--require-kubeconfig`选项后才从配置文件中读取 kube-apiserver 的地址，否则 kubelet 启动后将找不到 kube-apiserver \(日志中提示未找到 API Server），`kubectl get nodes`不会返回对应的 Node 信息;
* `--cluster-dns`指定 kubedns 的 Service IP\(可以先分配，后续创建 kubedns 服务时指定该 IP\)，`--cluster-domain`指定域名后缀，这两个参数同时指定后才会生效；
* `--cluster-domain`指定 pod 启动时`/etc/resolve.conf`文件中的`search domain`，起初我们将其配置成了`cluster.local.`，这样在解析 service 的 DNS 名称时是正常的，可是在解析 headless service 中的 FQDN pod name 的时候却错误，因此我们将其修改为`cluster.local`，去掉嘴后面的 ”点号“ 就可以解决该问题，关于 kubernetes 中的域名/服务名称解析请参见我的另一篇文章。
* `--kubeconfig=/etc/kubernetes/kubelet.kubeconfig`中指定的`kubelet.kubeconfig`文件在第一次启动kubelet之前并不存在，请看下文，当通过CSR请求后会自动生成`kubelet.kubeconfig`文件，如果你的节点上已经生成了`~/.kube/config`
  文件，你可以将该文件拷贝到该路径下，并重命名为`kubelet.kubeconfig`，所有node节点可以共用同一个kubelet.kubeconfig文件，这样新添加的节点就不需要再创建CSR请求就能自动添加到kubernetes集群中。同样，在任意能够访问到kubernetes集群的主机上使用`kubectl --kubeconfig`命令操作集群时，只要使用`~/.kube/config`文件就可以通过权限认证，因为这里面已经有认证信息并认为你是admin用户，对集群拥有所有权限。
* `KUBELET_POD_INFRA_CONTAINER`是基础镜像容器，这里我用的是私有镜像仓库地址，
  **大家部署的时候需要修改为自己的镜像**我上传了一个到时速云上，可以直接`docker pull index.tenxcloud.com/jimmy/pod-infrastructure`下载。

### 启动kublet {#启动kublet}

```
systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet
systemctl status kubelet
```

### 通过 kublet 的 TLS 证书请求 {#通过-kublet-的-tls-证书请求}

kubelet 首次启动时向 kube-apiserver 发送证书签名请求，必须通过后 kubernetes 系统才会将该 Node 加入到集群。

查看未授权的 CSR 请求

**首先要登陆到master 节点 去执行**

```
kubectl get csr

#NAME        AGE       REQUESTOR           CONDITION
#csr-hbhjz   1m        kubelet-bootstrap   Pending


kubectl get nodes
#NAME           STATUS    AGE       VERSION
#10.72.84.161   Ready     1m        v1.6.0

# approve 获得到的csr name
kubectl certificate approve csr-hbhjz
#certificatesigningrequest "csr-hbhjz" approved

kubectl get nodes
#NAME           STATUS    AGE       VERSION
#10.72.84.161   Ready     2m        v1.6.0
```

注意：假如你更新kubernetes的证书，只要没有更新`token.csv`，当重启kubelet后，该node就会自动加入到kuberentes集群中，而不会重新发送`certificaterequest`，也不需要在master节点上执行`kubectl certificate approve`操作。前提是不要删除node节点上的`/etc/kubernetes/ssl/kubelet*`和`/etc/kubernetes/kubelet.kubeconfig`文件。否则kubelet启动时会提示找不到证书而失败。

## 配置 kube-proxy {#配置-kube-proxy}

**创建 kube-proxy 的service配置文件  **

文件： 文件路径

`/usr/lib/systemd/system/kube-proxy.service`

```
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/kube-proxy \\
        \$KUBE_LOGTOSTDERR \\
        \$KUBE_LOG_LEVEL \\
        \$KUBE_MASTER \\
        \$KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


EOF
```

kube-proxy配置文件`/etc/kubernetes/proxy`

```
cat > /etc/kubernetes/proxy << EOF

###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=${IP} --hostname-override=${IP} --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"

EOF
```

* `--hostname-override`参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则；

* kube-proxy 根据`--cluster-cidr`判断集群内部和外部流量，指定`--cluster-cidr`或`--masquerade-all`选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；

* `--kubeconfig`指定的配置文件嵌入了 kube-apiserver 的地址、用户名、证书、秘钥等请求和认证信息；

* 预定义的 RoleBinding`cluster-admin`将User`system:kube-proxy`与 Role`system:node-proxier`绑定，该 Role 授予了调用  
  `kube-apiserver`Proxy 相关 API 的权限；

### 启动 kube-proxy {#启动-kube-proxy}

```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy
systemctl status kube-proxy -l
```



