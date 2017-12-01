# 部署node节点 {#部署node节点}

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





