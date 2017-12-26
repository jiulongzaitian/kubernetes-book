# kubectl 命令



```
张杰
```





## kubectl 原始方式访问api

```
kubectl get --raw=/apis/extensions/v1beta1 --recursive |python -m json.tool    
```



