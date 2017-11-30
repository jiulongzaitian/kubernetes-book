# 安装kubectl命令行工具 {#安装kubectl命令行工具}

本文档介绍下载和配置 kubernetes 集群命令行工具 kubelet 的步骤。

## 下载 kubectl {#下载-kubectl}

```
mkdir ~/sftp
cd ~/sftp

wget https://dl.k8s.io/v1.6.0/kubernetes-client-linux-amd64.tar.gz

tar -xzvf kubernetes-client-linux-amd64.tar.gz

cp kubernetes/client/bin/kube* /usr/bin/

chmod a+x /usr/bin/kube*
```

## 创建 kubectl kubeconfig 文件 {#创建-kubectl-kubeconfig-文件}

```




```



