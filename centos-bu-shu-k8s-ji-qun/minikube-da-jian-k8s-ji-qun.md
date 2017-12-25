# minikube 搭建k8s 集群



```
zhangjie  j.zhang8@haihangyun.com
```





# 说明

```
https://github.com/kubernetes/minikube
```



## 获取

```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```



# 启动

```
minikube start --vm-driver=none --extra-config=kubelet.HealthzPort=10248 --extra-config=kubelet.CadvisorPort=4196 --extra-config=kubelet.CgroupDriver=systemd  --extra-config=kubelet.RuntimeCgroups=/systemd/system.slice --extra-con
fig=kubelet.KubeletCgroups=/systemd/system.slice
```



注意 --extra-config 的写法

