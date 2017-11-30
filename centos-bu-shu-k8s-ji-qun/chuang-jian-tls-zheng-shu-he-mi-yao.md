# 创建TLS证书和秘钥 {#创建tls证书和秘钥}

## 前言 {#前言}

执行下列步骤前建议你先阅读以下内容：

* [管理集群中的TLS](https://jimmysong.io/kubernetes-handbook/guide/managing-tls-in-a-cluster.html)
  ：教您如何创建TLS证书
* [kubelet的认证授权](https://jimmysong.io/kubernetes-handbook/guide/kubelet-authentication-authorization.html)
  ：向您描述如何通过认证授权来访问 kubelet 的 HTTPS 端点。
* [TLS bootstrap](https://jimmysong.io/kubernetes-handbook/guide/tls-bootstrapping.html)
  ：介绍如何为 kubelet 设置 TLS 客户端证书引导（bootstrap）。

**注意**：这一步是在安装配置kubernetes的所有步骤中最容易出错也最难于排查问题的一步，而这却刚好是第一步，万事开头难，不要因为这点困难就望而却步。

**如果您足够有信心在完全不了解自己在做什么的情况下能够成功地完成了这一步的配置，那么您可以尽管跳过上面的几篇文章直接进行下面的操作。**

`kubernetes`系统的各组件需要使用`TLS`证书对通信进行加密，本文档使用`CloudFlare`的 PKI 工具集[cfssl](https://github.com/cloudflare/cfssl)来生成 Certificate Authority \(CA\) 和其它证书；

**生成的 CA 证书和秘钥文件如下：**

* ca-key.pem
* ca.pem
* kubernetes-key.pem
* kubernetes.pem
* kube-proxy.pem
* kube-proxy-key.pem
* admin.pem
* admin-key.pem

**使用证书的组件如下：**

* etcd：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
* kube-apiserver：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
* kubelet：使用 ca.pem；
* kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem；
* kubectl：使用 ca.pem、admin-key.pem、admin.pem；
* kube-controller-manager：使用 ca-key.pem、ca.pem

**注意：以下操作都在 master 节点这台主机上执行，证书只需要创建一次即可，以后在向集群中添加新节点时只要将 /etc/kubernetes/ 目录下的证书拷贝到新节点上即可。**

## 安装`CFSSL` {#安装-cfssl}

**方式一：直接使用二进制源码包安装**

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

**方式二：使用go命令安装**

我们的系统中安装了Go，使用以下命令安装更快捷：

```
go get -u github.com/cloudflare/cfssl/cmd/...
ls ${GOPATH}/bin/cfssl*
#cfssl cfssl-bundle cfssl-certinfo cfssljson cfssl-newkey cfssl-scan
```

## 创建 CA \(Certificate Authority\) {#创建-ca-certificate-authority}

**创建 CA 配置文件**

```
mkdir -p /etc/kubernetes/ssl
cd  /etc/kubernetes/ssl

cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
# 根据config.json文件的格式创建如下的ca-config.json文件
# 过期时间设置成了 87600h
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

字段说明

* `ca-config.json`：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
* `signing`：表示该证书可用于签名其它证书；生成的 ca.pem 证书中`CA=TRUE`；
* `server auth`：表示client可以用该 CA 对server提供的证书进行验证；
* `client auth`：表示server可以用该CA对client提供的证书进行验证；

**创建 CA 证书签名请求**

创建`ca-csr.json`文件，内容如下：

```
cat >> ca-csr.json << EOF
{
  "CN": "kubernetes",
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

"CN"：

* `Common Name`，kube-apiserver 从证书中提取该字段作为请求的用户名 \(User Name\)；浏览器使用该字段验证网站是否合法；
* "O"：`Organization`，kube-apiserver 从证书中提取该字段作为请求用户所属的组 \(Group\)；

**生成 CA 证书和私钥**

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

#2017/11/30 10:36:35 [INFO] generating a new CA key and certificate from CSR
#2017/11/30 10:36:35 [INFO] generate received request
#2017/11/30 10:36:35 [INFO] received CSR
#2017/11/30 10:36:35 [INFO] generating key: rsa-2048
#2017/11/30 10:36:36 [INFO] encoded CSR
#2017/11/30 10:36:36 [INFO] signed certificate with serial number 197604499255491654775533598924379621153884625031

ls ca*
# ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

## 创建 kubernetes 证书 {#创建-kubernetes-证书}

创建 kubernetes 证书签名请求文件`kubernetes-csr.json`：

```
cat >> kubernetes-csr.json << EOF

{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.72.84.160",
      "10.72.84.161",
      "10.72.84.162",
      "10.72.84.166",
      "10.72.84.167",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
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

* 如果 hosts 字段不为空则需要指定授权使用该证书 **IP 或域名列表**，由于该证书后续被`etcd`集群和`kubernetes master`

集群使用，所以上面分别指定了`etcd`集群、`kubernetes master`集群的主机 IP 和`kubernetes`**服务的服务 IP **一般是`kube-apiserver`指定的`service-cluster-ip-range`网段的第一个IP，如 10.254.0.1。

* hosts 中的内容可以为空，即使按照上面的配置，向集群中增加新节点后也不需要重新生成证书。但是我们建议hosts 有对应的ip 和域名
* 注意node 和master 的ip

**生成 kubernetes 证书和私钥**

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

#2017/11/30 10:43:22 [INFO] generate received request
#2017/11/30 10:43:22 [INFO] received CSR
#2017/11/30 10:43:22 [INFO] generating key: rsa-2048
#2017/11/30 10:43:22 [INFO] encoded CSR
#2017/11/30 10:43:22 [INFO] signed certificate with serial number 99667346270617055242305906445836661452974846966
#2017/11/30 10:43:22 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
#websites. For more information see the Baseline Requirements for the Issuance and Management
#of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
#specifically, section 10.2.3 ("Information Requirements").

ls kubernetes*

#kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
```

## 创建 admin 证书 {#创建-admin-证书}

创建 admin 证书签名请求文件`admin-csr.json`：

```
cat >> admin-csr.json << EOF
{
  "CN": "admin",
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
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```



* 后续`kube-apiserver`使用`RBAC`对客户端\(如`kubelet`、`kube-proxy  Pod`\)请求进行授权；

* `kube-apiserver`预定义了一些`RBAC`使用的`RoleBindings`，如`cluster-admin`将 Group`system:masters`与 Role`cluster-admin`绑定，该 Role 授予了调用`kube-apiserver`的**所有 API**的权限；
* OU 指定该证书的 Group 为`system:masters`，`kubelet`使用该证书访问`kube-apiserver`时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的`system:masters`，所以被授予访问所有 API 的权限；

生成 admin 证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

#2017/11/30 10:47:31 [INFO] generate received request
#2017/11/30 10:47:31 [INFO] received CSR
#2017/11/30 10:47:31 [INFO] generating key: rsa-2048
#2017/11/30 10:47:31 [INFO] encoded CSR
#2017/11/30 10:47:31 [INFO] signed certificate with serial number 68417547798438240484255100221611557251169203089
#2017/11/30 10:47:31 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
#websites. For more information see the Baseline Requirements for the Issuance and Management
#of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
#specifically, section 10.2.3 ("Information Requirements").

ls admin*
#admin.csr  admin-csr.json  admin-key.pem  admin.pem

```

## 创建 kube-proxy 证书 {#创建-kube-proxy-证书}

创建 kube-proxy 证书签名请求文件`kube-proxy-csr.json`：



