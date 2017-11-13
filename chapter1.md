# 利用 kubeadm 在 Google Cloud Platform  搭建kubernetes 集群

## Create vm instance

1 登陆  [https://console.cloud.google.com](https://console.cloud.google.com) 创建2个VM实例： 分别命名 kube-master1\(master node\), kube-node1 \(compute node\)

2 操作系统选择centos7 x86-64

3 本地mac系统安装GCP SDK，使用 gcloud  命令，将来会使用gcloud compute ssh 命令登陆VM ：

登陆说明： https://cloud.google.com/compute/docs/instances/connecting-to-instance\#standardssh

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

login your vm use gcloud command, then change root account

```
#gcloud compute ssh {your vm instance name}
$ sudo su -
```

### 1 Set  IPv4 traffic

Set `/proc/sys/net/bridge/bridge-nf-call-iptables`to `1`by running

`sysctl net.bridge.bridge-nf-call-iptables=1`to pass bridged IPv4 traffic to iptables’ chains.

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
  `--pod-network-cidr=192.168.0.0/16`to kubeadm in
  .
* Calico works on`amd64`only.

```
# kubeadm init --pod-network-cidr=192.168.0.0/16

[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[init] Using Kubernetes version: v1.8.2
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[kubeadm] WARNING: starting in 1.8, tokens expire after 24 hours by default (if you require a non-expiring token use --token-ttl 0)
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [kube-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.142.0.2]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests"
[init] This often takes around a minute; or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 29.502114 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node kube-master as master by adding a label and a taint
[markmaster] Master kube-master tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: 8f4968.b94417f5daafbdf1
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 8f4968.********** {your muster ip}:6443 --discovery-token-ca-cert-hash sha256:2c30233aee03d8141ec1bb23f68932d5b8a7219515620ddf79771f95c413d4de
```

To make kubectl work for your non-root user, you might want to run these commands \(which is also a part of the`kubeadm init`output\):`mkdir -p $HOME/.kube sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config sudo chown $(id -u):$(id -g) $HOME/.kube/config`Alternatively, if you are the root user, you could run this:

`export KUBECONFIG=/etc/kubernetes/admin.conf`

**Note** : I suggest that you need to record this output into your sepcifical files.

### 2 install calico 

```
kubectl apply -f https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
```



### 3 check master node

```
# kubectl get nodes
NAME          STATUS     ROLES     AGE       VERSION
kube-master   NotReady   master    4m        v1.8.1


# kubectl get pod --all-namespaces
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE
kube-system   calico-etcd-8s6fl                          1/1       Running   0          5m
kube-system   calico-kube-controllers-6ff88bf6d4-zlrbd   1/1       Running   0          4m
kube-system   calico-node-7987n                          2/2       Running   1          5m
kube-system   etcd-kube-master                           1/1       Running   0          4m
kube-system   kube-apiserver-kube-master                 1/1       Running   0          4m
kube-system   kube-controller-manager-kube-master        1/1       Running   0          4m
kube-system   kube-dns-545bc4bfd4-cmk96                  0/3       Pending   0          11m
kube-system   kube-proxy-tsbb2                           1/1       Running   0          11m
kube-system   kube-scheduler-kube-master                 1/1       Running   0          4m
```

**Note**: kube-dns pod is pending 

## Join your node 

1. ssh your kube-node1 vm instance use gcloud command
2. Become root\(e.g. sudo su -\)
3. Run the command that was output by kubeadm init logs  
   For example:

   ```
   kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
   ```

4. The output should look something like:

   ```
   [kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
   [preflight] Running pre-flight checks
   [preflight] Starting the kubelet service
   [discovery] Trying to connect to API Server "master ip:6443"
   [discovery] Created cluster-info discovery client, requesting info from "https://10.142.0.2:6443"
   [discovery] Requesting info from "https://10.142.0.2:6443" again to validate TLS against the pinned public key
   [discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "10.142.0.2:6443"
   [discovery] Successfully established connection with API Server "10.142.0.2:6443"
   [bootstrap] Detected server version: v1.8.2
   [bootstrap] The server supports the Certificates API (certificates.k8s.io/v1beta1)


   Node join complete:
   * Certificate signing request sent to master and response
     received.
   * Kubelet informed of new secure connection details.


   Run 'kubectl get nodes' on the master to see this machine join.


   ```



## Troubleshooting {#troubleshooting}

  ssh your master node: kube-master to check node status

```
#kubectl get nodes
NAME          STATUS     ROLES     AGE       VERSION
kube-master   Ready      master    20m       v1.8.1
kube-node1    NotReady   <none>    6m        v1.8.1
```

**Note** kube-node1 is NotReady

ssh kube-node1 to find reason:

```
#systemctl status kubelet -l
```

**Note** if  find errors , such as



```
failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container "/system.slice/docker.service"
```



you need to change /etc/systemd/system/kubelet.service.d/10-kubeadm.conf configfile , add this config to KUBELET\_CGROUP\_ARGS  such as:

```
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"
```

then restart kubelet



Note if find errors ,such as:

```
Nov 01 07:55:56 kube-node1 kubelet[25103]: W1101 07:55:56.905760   25103 cni.go:196] Unable to update cni config: No networks found in /etc/cni/net.d
Nov 01 07:55:56 kube-node1 kubelet[25103]: E1101 07:55:56.905958   25103 kubelet.go:2095] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
Nov 01 07:56:01 kube-node1 kubelet[25103]: W1101 07:56:01.907147   25103 cni.go:196] Unable to update cni config: No networks found in /etc/cni/net.d
Nov 01 07:56:01 kube-node1 kubelet[25103]: E1101 07:56:01.907741   25103 kubelet.go:2095] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
Nov 01 07:56:06 kube-node1 kubelet[25103]: W1101 07:56:06.908871   25103 cni.go:196] Unable to update cni config: No networks found in /etc/cni/net.d
Nov 01 07:56:06 kube-node1 kubelet[25103]: E1101 07:56:06.909014   25103 kubelet.go:2095] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
Nov 01 07:56:11 kube-node1 kubelet[25103]: W1101 07:56:11.910306   25103 cni.go:196] Unable to update cni config: No networks found in /etc/cni/net.d
Nov 01 07:56:11 kube-node1 kubelet[25103]: E1101 07:56:11.910846   25103 kubelet.go:2095] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
Nov 01 07:56:16 kube-node1 kubelet[25103]: W1101 07:56:16.912128   25103 cni.go:196] Unable to update cni config: No networks found in /etc/cni/net.d
Nov 01 07:56:16 kube-node1 kubelet[25103]: E1101 07:56:16.912258   25103 kubelet.go:2095] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

maybe lack cni config\(10-calico.conf  calico-kubeconfig\) in /etc/cni/net.d/ 

you need copy this two files from kube-master vm install . then ssh to kube-master vm instance, kube-node1 is ready

```
# kubectl get nodes
NAME          STATUS    ROLES     AGE       VERSION
kube-master   Ready     master    11m       v1.8.1
kube-node1    Ready     <none>    10m       v1.8.1
```













## Tear Down

To undo what kubeadm did, you should first[drain the node](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#drain)and make sure that the node is empty before shutting it down.

Talking to the master with the appropriate credentials, run:

```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```

Then, on the node being removed, reset all kubeadm installed state:

```
kubeadm reset
```



