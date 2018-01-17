# kubernetes 问题汇总

---

```
张杰
```





## 网络问题

1 kubelet 使用 cni bridge模式时候，一般会用vethpair，一般情况下，docker 不会创建 symlink:我们要想使用 ip netns 命令，

则首先需要创建软连接

```
# (as root)
pid=$(docker inspect -f '{{.State.Pid}}' ${container_id})
mkdir -p /var/run/netns/
ln -sfT /proc/$pid/ns/net /var/run/netns/$container_id
```

```
# 才能用ip netns 查看

# ip netns list
```



