# minikube 测试kubernetes ingress

```
张杰
```

---

## 使用minikube 搭建本地kubernetes环境

```
https://github.com/kubernetes/minikube


# 安装docker 
# 启动
minikube start --extra-config=kubelet.RuntimeCgroups=/systemd/system.slice --extra-config=kubelet.KubeletCgroups=/systemd/system.slice  --extra-config=kubelet.CgroupDriver=systemd --vm-driver=none

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

**注意： jiulongzaitian/test:1 镜像需要docker login，具体 kubernetes 使用pullimageSecrets 拉取镜像，请参照： **[**正确使用pullimageSecret **](/centos-bu-shu-k8s-ji-qun/zheng-que-shi-yong-pullimagesecret.md)

## 创建test svc 文件

```
cat > svc.yaml << EOF

apiVersion: v1
kind: Service
metadata:
  labels:
    run: test
  name: test
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    run: test
  type: NodePort

EOF
```

## 创建ing 文件

```
cat > ing.yaml << EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - http:
        paths:
          - backend:
              serviceName: test
              servicePort: 80
            path: /
EOF
```

注意： annotations 里如果没有 ingress.kubernetes.io/ssl-redirect: "false"   annotations,  则会报错：

```
<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.11.12</center>
</body>
</html>
```

## 部署ingress Nginx controller

参照：[https://github.com/kubernetes/ingress-nginx/blob/master/deploy/README.md](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/README.md)

```
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml \
    | kubectl apply -f -

# use RBAC

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml \
    | kubectl apply -f -
```

## 创建资源

```
kubectl create - f .
```

## 查看资源

```
kubectl get pod test
#NAME      READY     STATUS    RESTARTS   AGE
#test      1/1       Running   1          2h


kubectl get svc test
#NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
#test      NodePort   10.101.124.30   <none>        80:30506/TCP   2h


kubectl get ing test
#NAME      HOSTS     ADDRESS      PORTS     AGE
#test      *         10.146.0.7   80        22m

# 这时候 执行 
curl 10.146.0.7:80

#则会出现 hello world 字段，这是pod 里的服务产生的
```



