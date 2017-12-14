# 设置pod的Service Accounts

```
作者：苏晓林 xiaol.su@haihangyun.com
```

service account为在pod中运行的进程提供标识。

本文是service account的使用介绍。查看更多内容可以点击[_Cluster Admin Guide to Service Accounts_](https://kubernetes.io/docs/admin/service-accounts-admin/)。

注意：本文介绍了service account在kubernetes集群（推荐配置）中如何运行。若集群管理员对行为进行了自定义，本文可能不适用。

当你访问kubernetes集群时（例如使用`kubectl`），apiserver将对你使用的用户进行认证（通常帐户为`admin`，除非你修改了集群的默认配置）。当pod中的进程访问apiserver时，它们将会通过service account进行认证（如`default`）。

## 使用Default Service Account访问API server. {#use-the-default-service-account-to-access-the-api-server}

当创建pod时，若没有指定service account，将会在当前namespace中自动分配`default`service account。当你查看该pod的详细配置时（例，可以使用命令`kubectl get pods/podname -o yaml`），可以看到`spec.serviceAccountName`字段已经[automatically set](https://kubernetes.io/docs/user-guide/working-with-resources/#resources-are-automatically-modified)。

可以通过自动设置的service account从pod中访问API。查看[Accessing the Cluster](https://kubernetes.io/docs/user-guide/accessing-the-cluster/#accessing-the-api-from-a-pod)了解更多。service account的访问权限取决于[authorization plugin and policy](https://kubernetes.io/docs/admin/authorization/#a-quick-note-on-service-accounts)中的设置。

在1.6版本以上，可以通过设置`automountServiceAccountToken: false`取消service account的自动认证。

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

在1.6版本以上，也可以为指定的pod设置取消service account的自动认证。

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

pod spec中`automountServiceAccountToken`的优先级较高。

## 使用多个Service Accounts

每个namespace有都默认的service account：`default`。可以通过以下命令查看所有serviceAccount。

```
$ kubectl get serviceAccounts
NAME      SECRETS    AGE
default   1          1d
```

可以通过如下命令创建新的ServiceAccount

```
$ cat > /tmp/serviceaccount.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF
$ kubectl create -f /tmp/serviceaccount.yaml
serviceaccount "build-robot" created
```

查看ServiceAccount的完整配置

```
$ kubectl get serviceaccounts/build-robot -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-06-16T00:12:59Z
  name: build-robot
  namespace: default
  resourceVersion: "272500"
  selfLink: /api/v1/namespaces/default/serviceaccounts/build-robot
  uid: 721ab723-13bc-11e5-aec2-42010af0021e
secrets:
- name: build-robot-token-bvbk5
```

可以看到自动为该ServiceAccount生成了token

可以通过授权插件来[设置服务帐户的权限](https://kubernetes.io/docs/admin/authorization/#a-quick-note-on-service-accounts)。

通过设置pod的`spec.serviceAccountName`字段来指定service account。service account需要在pod创建之前就已经存在，否则会创建pod失败。以创建的pod无法更改service account。

删除service account

```
$ kubectl delete serviceaccount/build-robot
```

## 手动创建service account API token

假设已经存在上例中的service account：build-robot 。下面手动创建一个secret。

```
$ cat > /tmp/build-robot-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
EOF
$ kubectl create -f /tmp/build-robot-secret.yaml
secret "build-robot-secret" created
```

当前创建的secret的token为service account：build-robot 的token。

token controller会清除所有不存在service accounts的token。

```
$ kubectl describe secrets/build-robot-secret
Name:           build-robot-secret
Namespace:      default
Labels:         <none>
Annotations:    kubernetes.io/service-account.name=build-robot
                kubernetes.io/service-account.uid=da68f9c6-9d26-11e7-b84e-002dc52800da

Type:   kubernetes.io/service-account-token

Data
====
ca.crt:         1338 bytes
namespace:      7 bytes
token:          ...
```

注意：上面token的内容已省略

## 为service account添加ImagePullSecrets

首先，创建一个imagePullSecret，创建方法点击[这里](https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod)。然后查看imagePullSecret。

```
$ kubectl get secrets myregistrykey
NAME             TYPE                              DATA    AGE
myregistrykey    kubernetes.io/.dockerconfigjson   1       1d
```

然后，修改默认service account使用刚创建的imagePullSecret。

```
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "myregistrykey"}]}'
```

手动修改方法：

```
$ kubectl get serviceaccounts default -o yaml > ./sa.yaml
$ cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  resourceVersion: "243024"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
$ vi sa.yaml
[editor session not shown]
[delete line with key "resourceVersion"]
[add lines with "imagePullSecret:"]
$ cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
imagePullSecrets:
- name: myregistrykey
$ kubectl replace serviceaccount default -f ./sa.yaml
serviceaccounts/default
```

现在，所有当前namesapce下新创建的pod，都会拥有imagePullSecrets：myregistrykey

```
spec:
  imagePullSecrets:
  - name: myregistrykey
```



