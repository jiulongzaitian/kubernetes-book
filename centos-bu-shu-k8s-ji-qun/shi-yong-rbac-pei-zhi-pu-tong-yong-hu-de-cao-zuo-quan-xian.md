# 创建用户认证授权的kubeconfig文件 {#创建用户认证授权的kubeconfig文件}

张杰  j.zhang8@haihangyun.com

---

当我们安装好集群后，如果想要把 kubectl 命令交给用户使用，就不得不对用户的身份进行认证和对其权限做出限制。

下面以创建一个 qa 用户并将其绑定到 qa namespace 为例说明。

## 创建 CA 证书和秘钥 {#创建-ca-证书和秘钥}

**创建qa**`-csr.json`**文件**

```
mkdir -p /etc/kubernetes/ssl/rbac/qa
cd  /etc/kubernetes/ssl/rbac/qa
cat > qa-csr.json << EOF
{
  "CN": "qa",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

在[创建 TLS 证书和秘钥](/centos-bu-shu-k8s-ji-qun/chuang-jian-tls-zheng-shu-he-mi-yao.md)一节中我们将生成的证书和秘钥放在了所有节点的`/etc/kubernetes/ssl`目录下，下面我们再在 master 节点上为 qa 创建证书和秘钥，在`/etc/kubernetes/ssl/rbac/qa`目录下执行以下命令：

执行该命令前请先确保该目录下已经包含如下文件：

```
ca-config.json  ca-key.pem  ca.pem    qa-csr.json
```

我们可以设置软连接，将pem 软连接过来

```
cd /etc/kubernetes/ssl/rbac/qa
ln -s ../../ca-config.json ca-config.json
ln -s ../../ca-key.pem ca-key.pem
ln -s ../../ca.pem ca.pem

#创建 config dir
mkdir -p config

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes qa-csr.json | cfssljson -bare qa
```

## 创建 kubeconfig 文件 {#创建-kubeconfig-文件}

```
cd /etc/kubernetes/ssl/rbac/qa

# 需要IP 这个环境变量
if [ "$IP" == "" ]; then echo "NEED IP ENV"; fi

# 设置集群参数
export KUBE_APISERVER="https://${IP}:6443"

kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=qa.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials qa \
--client-certificate=/etc/kubernetes/ssl/rbac/qa/qa.pem \
--client-key=/etc/kubernetes/ssl/rbac/qa/qa-key.pem \
--embed-certs=true \
--kubeconfig=qa.kubeconfig

# 设置上下文参数
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=qa \
--namespace=qa \
--kubeconfig=qa.kubeconfig

# 设置默认上下文
kubectl config use-context kubernetes --kubeconfig=qa.kubeconfig

# 会在 /etc/kubernetes/ssl/rbac/qa 目录下 生成 qa.kubeconfig 文件
cat qa.kubeconfig


```



## 创建 资源文件 {#创建-kubeconfig-文件}

## 

```
cd /etc/kubernetes/ssl/rbac/qa/config

#创建namespace yaml
cat > ns.yaml << EOF
apiVersion: v1
kind: Namespace
metadata:
  name: qa
  
EOF

# 创建namespace
kubectl create -f ns.yaml


# 创建 resourceQuota 资源限额文件
cat > resource.yaml << EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: qa-resources
  namespace: qa 
spec:
  hard:
    pods: "20"
    requests.cpu: "4000m"
    requests.memory: 30Gi
    limits.cpu: "4000m"
    limits.memory: 30Gi
EOF

# 20个pods， 4核 ， 30 G

# 创建resourceQuota
kubectl create -f resource.yaml


```



## 创建 role-binding  {#创建-kubeconfig-文件}

```
cat > role-binding.yaml << EOF
# 以下允许用户"qa"从"qa"命名空间
#  绑定 clusterroles admin  角色
# 目的是让qa 用户拥有qa namespace 的最高级管理权限
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin 
  namespace: qa 
subjects:
- kind: User
  name: qa 
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole 
  name: admin 
  apiGroup: rbac.authorization.k8s.io
EOF

# 创建 rolebinding
kubectl create -f role-binding.yaml


```

## 分发  qa.kubeconfig 文件 {#创建-kubeconfig-文件}

将qa.kubeconfig 配置文件 和对应的kubectl 二进制 发给需要的用户

使用者需要做以下操作

```
mkdir -p /root/.kube 
cp qa.kubeconfig /root/.kube/config

#然后 就可以使用对应的kubectl 进行操作了
```



