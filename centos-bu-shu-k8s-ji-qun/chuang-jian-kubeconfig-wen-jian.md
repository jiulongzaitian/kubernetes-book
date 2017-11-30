# 创建 kubeconfig 文件 {#创建-kubeconfig-文件}

---

注意：请先参考[安装kubectl命令行工具](/centos-bu-shu-k8s-ji-qun/an-zhuang-kubectl-ming-ling-xing-gong-ju.md)，先在 master 节点上安装 kubectl 然后再进行下面的操作。

`kubelet`、`kube-proxy`等 Node 机器上的进程与 Master 机器的`kube-apiserver`进程通信时需要认证和授权；

kubernetes 1.4 开始支持由`kube-apiserver`为客户端生成 TLS 证书的TLS Bootstrapping功能，这样就不需要为每个客户端生成证书了；该功能**当前仅支持为**`kubelet`生成证书；

因为我的master节点和node节点复用，所有在这一步其实已经安装了kubectl。参考[安装kubectl命令行工具](/centos-bu-shu-k8s-ji-qun/an-zhuang-kubectl-ming-ling-xing-gong-ju.md)。

以下操作只需要在master节点上执行，生成的`*.kubeconfig`文件可以直接拷贝到node节点的`/etc/kubernetes`目录下。

