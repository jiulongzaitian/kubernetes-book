# 创建TLS证书和秘钥 {#创建tls证书和秘钥}

```
作者：张杰  j.zhang8@haihangyun.com
```

---

## 前言 {#前言}

执行下列步骤前建议你先阅读以下内容：

* [管理集群中的TLS](https://jimmysong.io/kubernetes-handbook/guide/managing-tls-in-a-cluster.html)：教您如何创建TLS证书
* [kubelet的认证授权](https://jimmysong.io/kubernetes-handbook/guide/kubelet-authentication-authorization.html)：向您描述如何通过认证授权来访问 kubelet 的 HTTPS 端点。
* [TLS bootstrap](https://jimmysong.io/kubernetes-handbook/guide/tls-bootstrapping.html)：介绍如何为 kubelet 设置 TLS 客户端证书引导（bootstrap）。

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

```bash
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
cat > ca-csr.json << EOF
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

```shell
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
cat > kubernetes-csr.json << EOF

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

* hosts 中的内容可以为空，即使按照上面的配置，向集群中增加新节点后也不需要重新生成证书。但是我们建议hosts 里至少要包含etcd 集群的三个ip，因为后面的章节中etcd 集群使用的证书就是此证书
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
cat > admin-csr.json << EOF
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

```
cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
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

* CN 指定该证书的 User 为`system:kube-proxy`；

* `kube-apiserver`预定义的 RoleBinding`cluster-admin`将User`system:kube-proxy`与 Role`system:node-proxier`绑定，该 Role 授予了调用`kube-apiserver`Proxy 相关 API 的权限；

生成 kube-proxy 客户端证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy

#2017/11/30 10:51:12 [INFO] generate received request
#2017/11/30 10:51:12 [INFO] received CSR
#2017/11/30 10:51:12 [INFO] generating key: rsa-2048
#2017/11/30 10:51:12 [INFO] encoded CSR
#2017/11/30 10:51:12 [INFO] signed certificate with serial number 79574947778988512067987882646177426225975839744
#2017/11/30 10:51:12 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
#websites. For more information see the Baseline Requirements for the Issuance and Management
#of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
#specifically, section 10.2.3 ("Information Requirements").

ls kube-proxy*
#kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
```

## 校验证书 {#校验证书}

以 kubernetes 证书为例

### 使用`opsnssl`命令 {#使用-opsnssl-命令}

```
openssl x509  -noout -text -in  kubernetes.pem
```

结果

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            11:75:3d:06:41:a3:49:d3:f2:ab:53:0e:e8:6b:80:b6:59:e7:f3:f6
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=kubernetes
        Validity
            Not Before: Nov 30 02:38:00 2017 GMT
            Not After : Nov 28 02:38:00 2027 GMT
        Subject: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a0:19:39:a6:2c:46:f4:d1:8c:4d:3d:2d:2f:a8:
                    ca:97:0c:e6:7a:10:0e:17:d7:ac:9e:1f:c1:bc:a7:
                    db:fe:fc:d6:8f:17:6f:90:5f:94:ee:77:8e:c3:92:
                    a0:5e:cd:3b:14:d3:99:70:d2:d7:6f:fd:93:53:75:
                    39:29:65:8f:11:5f:d0:5d:42:d0:75:8d:96:cf:73:
                    85:81:6b:51:cf:5e:95:93:1e:95:a6:57:92:6d:34:
                    45:28:0e:37:4d:c8:c5:42:5d:5a:6f:66:f0:a0:08:
                    91:35:8c:9e:30:41:84:30:5e:27:9f:43:c3:20:03:
                    32:08:89:2d:9a:d4:c5:bc:8f:2e:79:5a:18:9a:d9:
                    85:76:da:a8:d5:86:27:1e:76:c3:b5:47:f8:60:37:
                    16:e0:7f:55:5a:76:fb:19:4c:de:a5:ad:0d:5f:1e:
                    e6:be:7f:54:a5:9a:2b:76:1e:b1:9c:25:41:47:35:
                    51:c8:9a:e5:71:78:76:8a:e9:81:84:15:bb:30:da:
                    44:2f:9e:05:3c:f4:ef:c7:2b:29:7d:03:75:f9:d9:
                    54:40:34:8c:d0:61:a4:1b:e0:80:10:32:73:99:bc:
                    2a:81:46:0f:f2:41:50:f1:38:00:29:cf:1c:01:62:
                    51:1a:e2:a1:c5:1c:46:76:cb:4c:10:90:a2:87:64:
                    eb:3b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier: 
                60:F9:9D:4F:A9:C1:1A:F2:26:DA:3F:4C:5D:39:21:24:7E:88:A1:6E
            X509v3 Authority Key Identifier: 
                keyid:9A:A5:13:18:93:16:9A:52:12:76:5A:4A:87:E7:C9:77:C3:1F:E5:F2

            X509v3 Subject Alternative Name: 
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, IP Address:127.0.0.1, IP Address:10.72.84.160, IP Address:10.72.84.161, IP Address:10.72.84.162, IP Address:10.72.84.166, IP Address:10.72.84.167, IP Address:10.254.0.1
    Signature Algorithm: sha256WithRSAEncryption
         a9:84:4d:c0:d5:6b:f3:f2:a1:20:ef:f6:5a:94:c0:ef:4a:54:
         6f:a4:40:5b:e2:f5:f9:26:76:97:e4:d2:0c:bf:a8:43:67:fb:
         a4:05:0d:a7:ae:b4:0f:21:39:b3:37:55:af:a5:16:5d:2f:2a:
         87:64:01:02:54:48:b3:52:e9:21:64:b3:a4:0e:70:11:3e:ce:
         f5:f2:74:ad:0c:ae:b1:b8:81:17:26:1e:8d:4f:c2:2b:17:a0:
         19:7f:2d:6e:db:e4:bc:de:53:39:9e:54:31:6a:0e:fd:d4:77:
         92:0a:5c:af:02:7d:33:89:38:44:c5:fa:a4:e0:62:0c:57:e1:
         0e:ea:d5:49:6b:9a:59:9f:bc:4e:1e:b0:74:75:66:2d:bc:ad:
         6d:6a:6b:e0:ef:01:04:bb:b8:9c:54:bc:d8:ac:2c:5c:b1:39:
         28:70:e0:ba:d0:b6:68:e0:8d:c8:b2:01:b6:56:a7:f2:7b:41:
         c3:6e:2c:a9:09:0c:2d:a9:58:19:a4:47:2a:37:7c:11:e5:06:
         91:bc:98:84:fc:4d:06:20:5c:e1:49:b3:ac:43:f2:14:45:d0:
         97:77:e6:07:e0:98:d3:69:65:9e:ac:98:8b:21:2d:ea:28:13:
         15:d3:a0:d9:89:de:7a:6d:98:2e:ea:1a:0c:dc:e1:00:42:3c:
         9e:4d:9f:86
```

确认

* `Issuer`字段的内容和`ca-csr.json`一致；
* 确认`Subject`字段的内容和`kubernetes-csr.json`一致；
* 确认`X509v3 Subject Alternative Name`字段的内容和`kubernetes-csr.json`一致；
* 确认`X509v3 Key Usage、Extended Key Usage`字段的内容和`ca-config.json`中`kubernetes`profile 一致；

使用`cfssl-certinfo`命令

```
cfssl-certinfo -cert kubernetes.pem
```

结果

```
{
  "subject": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "issuer": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "serial_number": "99667346270617055242305906445836661452974846966",
  "sans": [
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local",
    "127.0.0.1",
    "10.72.84.160",
    "10.72.84.161",
    "10.72.84.162",
    "10.72.84.166",
    "10.72.84.167",
    "10.254.0.1"
  ],
  "not_before": "2017-11-30T02:38:00Z",
  "not_after": "2027-11-28T02:38:00Z",
  "sigalg": "SHA256WithRSA",
  "authority_key_id": "9A:A5:13:18:93:16:9A:52:12:76:5A:4A:87:E7:C9:77:C3:1F:E5:F2",
  "subject_key_id": "60:F9:9D:4F:A9:C1:1A:F2:26:DA:3F:4C:5D:39:21:24:7E:88:A1:6E",
  "pem": "-----BEGIN CERTIFICATE-----\nMIIEkTCCA3mgAwIBAgIUEXU9BkGjSdPyq1MO6GuAtlnn8/YwDQYJKoZIhvcNAQEL\nBQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0Jl\naUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwpr\ndWJlcm5ldGVzMB4XDTE3MTEzMDAyMzgwMFoXDTI3MTEyODAyMzgwMFowZTELMAkG\nA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxDDAK\nBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwprdWJlcm5ldGVz\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAoBk5pixG9NGMTT0tL6jK\nlwzmehAOF9esnh/BvKfb/vzWjxdvkF+U7neOw5KgXs07FNOZcNLXb/2TU3U5KWWP\nEV/QXULQdY2Wz3OFgWtRz16Vkx6VpleSbTRFKA43TcjFQl1ab2bwoAiRNYyeMEGE\nMF4nn0PDIAMyCIktmtTFvI8ueVoYmtmFdtqo1YYnHnbDtUf4YDcW4H9VWnb7GUze\npa0NXx7mvn9UpZordh6xnCVBRzVRyJrlcXh2iumBhBW7MNpEL54FPPTvxyspfQN1\n+dlUQDSM0GGkG+CAEDJzmbwqgUYP8kFQ8TgAKc8cAWJRGuKhxRxGdstMEJCih2Tr\nOwIDAQABo4IBNzCCATMwDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUF\nBwMBBggrBgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBRg+Z1PqcEa8iba\nP0xdOSEkfoihbjAfBgNVHSMEGDAWgBSapRMYkxaaUhJ2WkqH58l3wx/l8jCBswYD\nVR0RBIGrMIGoggprdWJlcm5ldGVzghJrdWJlcm5ldGVzLmRlZmF1bHSCFmt1YmVy\nbmV0ZXMuZGVmYXVsdC5zdmOCHmt1YmVybmV0ZXMuZGVmYXVsdC5zdmMuY2x1c3Rl\ncoIka3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FshwR/AAABhwQK\nSFSghwQKSFShhwQKSFSihwQKSFSmhwQKSFSnhwQK/gABMA0GCSqGSIb3DQEBCwUA\nA4IBAQCphE3A1Wvz8qEg7/ZalMDvSlRvpEBb4vX5JnaX5NIMv6hDZ/ukBQ2nrrQP\nITmzN1WvpRZdLyqHZAECVEizUukhZLOkDnARPs718nStDK6xuIEXJh6NT8IrF6AZ\nfy1u2+S83lM5nlQxag791HeSClyvAn0ziThExfqk4GIMV+EO6tVJa5pZn7xOHrB0\ndWYtvK1tamvg7wEEu7icVLzYrCxcsTkocOC60LZo4I3IsgG2Vqfye0HDbiypCQwt\nqVgZpEcqN3wR5QaRvJiE/E0GIFzhSbOsQ/IURdCXd+YH4JjTaWWerJiLIS3qKBMV\n06DZid56bZgu6hoM3OEAQjyeTZ+G\n-----END CERTIFICATE-----\n"
}
```

## 分发证书 {#分发证书}

将生成的证书和秘钥文件（后缀名为`.pem`）拷贝到所有机器的`/etc/kubernetes/ssl`目录下备用；

