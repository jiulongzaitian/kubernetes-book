```
作者：李昂 邮箱：liang@haihangyun.com
```


### 介绍
---
从Kubernetes 1.3起，DNS是使用插件管理器集群附件自动启动的内置服务。
Kubernetes DNS在集群上设置DNS Pod和服务，并配置kubelet以告诉单个容器使用DNS服务的IP来解析DNS名称。

### 哪些会有DNS域名

---
群集中定义的每个服务（包括DNS服务器本身）都被分配一个DNS名称。默认情况下，客户端Pod的DNS搜索列表将包括Pod自己的名称空间和群集的默认域。举例说明：

在Kubernetes命名空间bar中假设一个名为foo的服务。在命名空间bar中运行的Pod可以通过简单地为foo执行DNS查询来查找该服务。在名称空间quux中运行的Pod可以通过为foo.bar执行DNS查询来查找此服务。

### 支持的DNS概要

---
以下部分详细介绍支持的受支持的记录类型和布局。任何其他布局或名称或查询碰巧工作被视为实施细节，并可能会更改，恕不另行通知。有关更新的规范，请参阅Kubernetes基于DNS的服务发现。

#### Services
##### A记录

“Normal”（非headless）Services会为my-svc.my-namespace.svc.cluster.local的名称分配一个DNS A记录。这解析为服务的群集IP。

“headless”（没有cluster IP）Service也为my-svc.my-namespace.svc.cluster.local形式的名称分配了DNS A记录。与normal Service不同，其会解析到了由服务选择的一组Pod IP。客户端会消费该集合，或者使用集合中的round-robin选择IP。

##### SRV记录
为normal或headless Services的一部分的命名端口创建SRV记录。对于每个指定的端口，SRV记录的形式是_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster.local。对于normal Service，这将解析为端口号和CNAME：my-svc.my-namespace.svc.cluster.local。对于headless Services，这将解析到多个应答，每个支持Service的pod都有一个应答，并且包含端口号和Pod的CNAME有如下形式auto-generated-name.my-svc.my-namespace.svc .cluster.local。

##### 向后兼容
早期版本的kube-dns使得my-svc.my-namespace.cluster.local（“svc”级别稍后添加）的名称成为名称。这不再被支持。

#### Pod
##### A记录
当开启时，Pod会以pod-ip-address.my-namespace.pod.cluster.local的形式分配一个DNS A记录。

例如，名称空间默认情况下，pod IP地址为1.2.3.4的在DNS名字为cluster.local将具有一个条目：1-2-3-4.default.pod.cluster.local。

##### A记录和hostname基于pod的hostname和subdomain字段
当前创建一个pod时，其hostname是Pod的metadata.name值。

在v1.2中，用户可以指定Pod annotation pod.beta.kubernetes.io/hostname来指定Pod的主机名。 Pod annotation（如果指定的话）优先于Pod的名称，作为Pod的hostname。例如，给定带有 annotation pod.beta.kubernetes.io/hostname：my-pod-name的Pod，Pod将其hostname设置为“my-pod-name”。

在v1.3中，PodSpec有一个hostname字段，可以用来指定Pod的hostname。该字段值优先于pod.beta.kubernetes.io/hostname值。

v1.2引入了一个测试版功能，用户可以指定一个Pod annotation pod.beta.kubernetes.io/subdomain来指定Pod的子域。例如，在命名空间“my-namespace”中，hostname annotation设置为“foo”的Pod以及设置为“bar”的子域annotation将具有FQDN “foo.bar.my-namespace.svc.cluster.local”。

在v1.3中，PodSpec有一个子域名字段，可以用来指定Pod的子域名。此字段值优先于pod.beta.kubernetes.io/subdomain annotation。

示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo # Actually, no port is needed.
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox

```
如果在与pod相同的命名空间中存在与子域名相同的命名空间的headless Service，则群集的KubeDNS服务器也会返回Pod的完全限定主机名的A记录。给定主机名设置为“busybox-1”，子域设置为“default-subdomain”的Pod以及名称为“default-subdomain”的headless Service在同一个名称空间中，该Pod将自己的FQDN视为“busybox- 1.default-subdomain.my-namespace.svc.cluster.local”。 DNS以该名称提供A记录，指向Pod的IP。 “busybox1”和“busybox2”这两个容器都可以具有不同的A记录。

从Kubernetes v1.2开始，Endpoints对象也有annotation  endpoints.beta.kubernetes.io/hostnames-map。它的值是map [string（IP）] [endpoints.HostRecord]的json表示，例如：“{”10.245.1.6“：{HostName：”my-webserver“}}'。如果Endpoints用于headless Service，则使用格式... svc创建A记录。对于json示例，如果Endpoints是用于名为“bar”的headless Service，而其中一个端点的IP为“10.245.1.6”，则创建一个名为“my-webserver.bar.my-namespace.svc”的A记录.cluster.local“，A记录查找将返回”10.245.1.6“。这个Endpoints annotation通常不需要由最终用户来指定，而是可以被内部服务控制器用来提供上述特征。

在v1.3中，Endpoints对象可以指定任何端点的主机名及其IP地址。主机名字段优先于可能通过endpoints.beta.kubernetes.io/hostnames-map注释指定的主机名称。

在v1.3中，不赞成使用以下annotations ：pod.beta.kubernetes.io/hostname，pod.beta.kubernetes.io/subdomain，endpoints.beta.kubernetes.io/hostnames-map。

### 如何测试

---
#### 创建一个简单Pod用于测试环境
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```
#### 等待这个Pod运行起来

```
NAME      READY     STATUS    RESTARTS   AGE
busybox   1/1       Running   0          <some-time>
```

#### 验证DNS是否工作
```
kubectl exec -ti busybox -- nslookup kubernetes.default

```
```
Server:    10.0.0.10
Address 1: 10.0.0.10

Name:      kubernetes.default
Address 1: 10.0.0.1
```
如果你看到如上所示，则DNS工作正常。

#### DNS策略
默认情况下，pod的DNS策略是“ClusterFirst”。所以使用hostNetwork运行的pod无法解析DNS名称。要与hostNetwork一起设置DNS选项，您应该明确指定“ClusterFirstWithHostNet”的DNS策略。更新busybox.yaml如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

### Kubernetes联邦（支持多区域）

---
版本1.3引入了用于多区域Kubernetes安装的群集联邦支持。这需要对Kubernetes集群DNS服务器处理DNS查询的方式进行一些较小的（向后兼容）更改，以便于查找联邦服务（跨多个Kubernetes集群）。有关Cluster Federation和多区域支持的更多详细信息，请参阅“Cluster Federation管理员指南”。

### DNS如何工作的

---
正在运行的Kubernetes DNS pod包含3个容器 - kubedns，dnsmasq和一个名为healthz的健康检查。 kubedns进程监视Kubernetes主服务和端点的变化，并维护内存查找结构来为DNS请求提供服务。 dnsmasq容器添加DNS缓存以提高性能。 healthz容器在执行双健康检查（对于dnsmasq和kubedns）时提供单个运行状况检查端点。

DNS pod作为具有静态IP的Kubernetes服务公开。分配后，kubelet会将使用--cluster-dns = 10.0.0.10标志配置的DNS传递给每个容器。

DNS名称也需要域名。本地域是可配置的，在kubelet中使用--cluster-domain设置。

Kubernetes集群DNS服务器（基于SkyDNS库）支持正向查找（A记录），服务查找（SRV记录）和反向IP地址查找（PTR记录）。

### 从节点继承DNS

---
运行pod时，kubelet将预先安装集群DNS服务器，并搜索节点自己的DNS设置路径。如果节点能够解析特定于更大环境的DNS名称，那么Pod也应该能够。有关警告，请参阅下面的“已知问题”。

如果你不想要这个，或者你想要一个不同的DNS配置的Pod，你可以使用kubelet的--resolv-conf标志。设置为“”意味着pod不会继承DNS。将其设置为有效的文件路径意味着kubelet将使用此文件而不是/etc/resolv.conf用于DNS继承。
