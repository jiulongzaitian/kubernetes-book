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



