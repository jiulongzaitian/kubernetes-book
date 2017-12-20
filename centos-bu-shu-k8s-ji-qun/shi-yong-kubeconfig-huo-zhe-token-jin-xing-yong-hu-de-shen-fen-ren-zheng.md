# 使用kubeconfig 或者token 进行用户的身份认证

```
张杰             j.zhang8@haihangyun.com
```

---

在开启了 TLS 的集群中，每当与集群交互的时候少不了的是身份认证，使用 kubeconfig（即证书） 和 token 两种认证方式是最简单也最通用的认证方式，在 dashboard 的登录功能就可以使用这两种登录功能。

下文分两块以示例的方式来讲解两种登陆认证方式：

* 为 admin 用户创建 kubeconfig 文件
* 为集群的管理员（拥有所有命名空间的 amdin 权限）创建 token

如何生成`kubeconfig`文件请参考  [创建用户认证授权的kubeconfig文件](/centos-bu-shu-k8s-ji-qun/shi-yong-rbac-pei-zhi-pu-tong-yong-hu-de-cao-zuo-quan-xian.md)

**注意**我们生成的 kubeconfig 文件中没有 token 字段，需要手动添加该字段。

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t ***
    server: https://135.100.6.43:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: admin
  name: kubernetes
current-context: kubernetes
kind: Config
preferences: {}
users:
- name: admin
  user:
    as-user-extra: {}
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t ***
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUl= ***

```

对于访问 dashboard 时候的使用 kubeconfig 文件必须追到`token`字段，否则认证不会通过。而使用 kubectl 命令时的用的 kubeconfig 文件则不需要包含`token`字段。

### 生成 token {#生成-token}

需要创建一个admin用户并授予admin角色绑定，使用下面的yaml文件创建admin用户并赋予他管理员权限，然后可以通过token访问kubernetes

```
mkdir -p ~/sftp/yaml/dashborad
cd   ~/sftp/yaml/dashborad

cat > admin-role.yaml << EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile

EOF
```

然后执行下面的命令创建 serviceaccount 和角色绑定，对于其他命名空间的其他用户只要修改上述 yaml 中的`name`和`namespace`字段即可：

```
kubectl create  -f  admin-role.yaml
```

创建完成后获取secret和token的值。



```
kubectl -n kube-system get secret|grep admin-token
# admin-token-frffk                  kubernetes.io/service-account-token   3         13m

# 获取token的值
 kubectl -n kube-system describe secret admin-token-frffk
 
 
 # 
```

**注意**：yaml 输出里的那个 token 值是进行 base64 编码后的结果，一定要将 kubectl 的输出中的 token 值进行`base64`解码，在线解码工具[base64decode](https://www.base64decode.org/)，Linux 和 Mac 有自带的`base64`命令也可以直接使用，输入`base64`是进行编码，Linux 中`base64 -d`表示解码，Mac 中使用`base64 -D`。

我们使用了 base64 对其重新解码，因为 secret 都是经过 base64 编码的，如果直接使用 kubectl 中查看到的`token`值会认证失败，详见[secret 配置](https://jimmysong.io/kubernetes-handbook/guide/secret-configuration.html)。关于 JSONPath 的使用请参考[JSONPath 手册](https://kubernetes.io/docs/user-guide/jsonpath/)。

  


