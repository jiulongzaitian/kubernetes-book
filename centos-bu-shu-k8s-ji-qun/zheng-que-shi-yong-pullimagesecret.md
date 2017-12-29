# **正确使用pullimageSecret **

```
张杰
```



# 创建secret

**方式1：**

```
kubectl create secret docker-registry NAME --docker-username=user --docker-password=password --docker-email=email [--docker-server=string]
```

注意： 一般如果你是使用官方dockerhub 镜像仓库，--docker-server 可以不写



方式2：以dockerhub 镜像仓库为例

```
# 首先执行docker login 登陆,输入用户名和密码
docker login


SECRET=$(cat ~/.docker/config.json |base64 -w 0)


# 创建secret 文件

cat > jiu.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: jiu
data:
  .dockerconfigjson: ${SECRET}
type: kubernetes.io/dockerconfigjson
EOF

#创建secret 
kubectl create -f jiu.yaml



```



## POD 使用imageSecret



**方式1：**

```
cat > pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: test
  namespace: default
spec:
  imagePullSecrets:
    - name: jiu
  containers:
    - name: test
      image: jiulongzaitian/test:1
      imagePullPolicy: Always
      command:
        - sh
        - -c
        - /tmp/test



EOF
```

在spec 里设定 imagePullSecrets ，name 则为secret 对应的name



**方式2：**

```
# 设定对应namespace 的default serviceAccount
kubectl get sa default

#NAME      SECRETS   AGE
#default   1         3d


# 编辑default sa

kubectl edit sa default

# 增加 
#imagePullSecrets:
#  - name: jiu

# 注意 imagePullSecrets 与apiVersion 同级
```

例如：

```
apiVersion: v1
imagePullSecrets:
- name: jiu
kind: ServiceAccount
metadata:
  creationTimestamp: 2017-12-25T13:02:27Z
  name: default
  namespace: default
  resourceVersion: "126175"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: db799047-e973-11e7-9244-42010a920007
secrets:
- name: default-token-sdgn5
```

由于pod 的创建都会默认加载对应namespace 的sa，所以，imagePullSecrets 则会自动加载上去

