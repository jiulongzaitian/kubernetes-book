# 利用 kubeadm 在 Google Cloud Platform  搭建kubernetes 集群

## Create vm instance

1 登陆  [https://console.cloud.google.com](https://console.cloud.google.com) 创建2个VM实例： 分别命名 kube-master1\(master node\), kube-node1 \(compute node\)

2 操作系统选择centos7 x86-64

3 本地mac系统安装GCP SDK，使用 gcloud  命令，将来会使用gcloud compute ssh 命令登陆VM

4 add new yum repo

```
vim /etc/yum.repos.d/kubernetes.repo 

[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
```

## Install kubeadm on vm instance

### 1 Set  IPv4 traffic

Set `/proc/sys/net/bridge/bridge-nf-call-iptables `to `1 `by running

`sysctl net.bridge.bridge-nf-call-iptables=1 `to pass bridged IPv4 traffic to iptables’ chains.

### 2 install docker\(version 1.12\) on your all vm

On each of your machines, install Docker. Version v1.12 is recommended, but v1.11, v1.13 and 17.03 are known to work as well. Versions 17.06+  _might work _, but have not yet been tested and verified by the Kubernetes node team

```
# yum info docker
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: ftp.linux.ncsu.edu
 * epel: mirror.us.leaseweb.net
 * extras: repo1.ash.innoscale.net
 * updates: www.gtlib.gatech.edu
Installed Packages
Name        : docker
Arch        : x86_64
Epoch       : 2
Version     : 1.12.6
Release     : 61.git85d7426.el7.centos
Size        : 52 M
Repo        : installed
From repo   : extras
Summary     : Automates deployment of containerized applications
URL         : https://github.com/docker/docker
License     : ASL 2.0
Description : Docker is an open-source engine that automates the deployment of any
            : application as a lightweight, portable, self-sufficient container that will
            : run virtually anywhere.
            : 
            : Docker containers can encapsulate any payload, and will run consistently on
            : and between virtually any server. The same container that a developer builds
            : and tests on a laptop will run at scale, in production*, on VMs, bare-metal
            : servers, OpenStack clusters, public instances, or combinations of the above.
            
@ yum install docker -y



```

**Note** 

: Make sure that the cgroup driver used by kubelet is the same as the one used by Docker. 

```
# cat /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target
Wants=docker-storage-setup.service
Requires=docker-cleanup.timer

[Service]
Type=notify
NotifyAccess=all
EnvironmentFile=-/run/containers/registries.conf
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
Environment=GOTRACEBACK=crash
Environment=DOCKER_HTTP_HOST_COMPAT=1
Environment=PATH=/usr/libexec/docker:/usr/bin:/usr/sbin
ExecStart=/usr/bin/dockerd-current \
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY\
          $REGISTRIES
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
Restart=on-abnormal
MountFlags=slave
KillMode=process

[Install]
WantedBy=multi-user.target
```

Please Check:

```
--exec-opt native.cgroupdriver=systemd
```

it must be seem with kubelet config, seem cgroupdirver



### 3 install kubeadm\(version 1.8 \) on your all vm

```
# yum info kubeadm
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: ftp.linux.ncsu.edu
 * epel: mirror.us.leaseweb.net
 * extras: repo1.ash.innoscale.net
 * updates: centos.vwtonline.net
Installed Packages
Name        : kubeadm
Arch        : x86_64
Version     : 1.8.1
Release     : 0
Size        : 89 M
Repo        : installed
From repo   : kubernetes
Summary     : Command-line utility for administering a Kubernetes cluster.
URL         : https://kubernetes.io
License     : ASL 2.0
Description : Command-line utility for administering a Kubernetes cluster.



# yum install kubeadm -y 

```



## Initializing Master

### 1 use calico network

**Note:**

* In order for Network Policy to work correctly, you need to pass
  `--pod-network-cidr=192.168.0.0/16 `to kubeadm in
  .
* Calico works on`amd64`only.

```
# kubeadm init --pod-network-cidr=192.168.0.0/16

```



