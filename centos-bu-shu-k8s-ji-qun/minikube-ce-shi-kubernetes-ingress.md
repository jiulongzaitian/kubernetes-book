# minikube 测试kubernetes ingress

```
张杰
```

---

## 使用minikube 搭建本地kubernetes环境

```
https://github.com/kubernetes/minikube
```

## 

## 开启ingress权限

```
minikube addons enable ingress
```



## 创建pod 文件

```
cat >pod.xml << EOF

apiVersion: v1
kind: Pod
metadata:
  name: test
  namespace: default
  labels:
    run: test
spec:
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

**注意： jiulongzaitian/test:1 镜像需要docker login，具体 kubernetes 使用pullimageSecrets 拉取镜像，请参照： 正确使用pullimageSecret **

