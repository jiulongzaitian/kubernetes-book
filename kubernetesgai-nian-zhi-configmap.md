# ConfigMap介绍

## ConfigMap的使命

在许多应用程序的实际配置中，常常需要通过命令行参数和环境变量进行混合的配置。复杂的配置会严重增加容器的部署和更新的难度、降低容器的可移植性。ConfigMap的出现就是为了解决该问题，通过ConfigMap将配置与镜像内容进行分离，镜像内容中不再包含需要动态变化的配置，全部放在ConfigMap中。从而使容器镜像具备了更好的可移植性，部署和更新也变得更容易。ConfigMap能够存储细粒度的信息（如单个属性）和粗粒度的信息（如整个配置文件或JSON对象），在容器运行时通过ConfigMap API资源将配置注入到容器中。

## ConfigMap概述

ConfigMap API资源中能够保存Pod中能够使用的key-value配置，也能够存储系统组件（如Controller）的配置数据。ConfigMap在一定程度上与Secrets有些类似，但是ConfigMap在设计能够更方便的支持不包含敏感信息的字符串。

ConfigMap示例：

```
kind: ConfigMap
apiVersion: v1
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: example-config
  namespace: default
data:
  example.property.1: hello
  example.property.2: world
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

在该示例中，data字段包含了细粒度的配置信息example.property.1和example.property.2，也包括粗粒度的信息example.property.file。

ConfigMap在Pod中的使用支持多种方法，如下：

1.设置环境变量的值

2.设置容器中的命令行参数

3.设置volume中的配置文件

用户和系统的组件都可以在ConfigMap中存储配置数据。

## 创建ConfigMap

创建ConfigMap的命令为`kubectl create configmap`，通过该命令可以直接通过目录、文件或key-value值的方式创建ConfigMap。

下面将对三种创建方式进行介绍：

### 目录创建

如下，在目录docs/user-guide/configmap/kubectl/中，存在配置game.properties和ui.properties：

```
$ ls docs/user-guide/configmap/kubectl/
game.properties
ui.properties

$ cat docs/user-guide/configmap/kubectl/game.properties
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30

$ cat docs/user-guide/configmap/kubectl/ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
```

通过目录创建ConfigMap来保存该目录下每个文件的内容：

```
$ kubectl create configmap game-config --from-file=docs/user-guide/configmap/kubectl
```

参数--from-file用来指向提供创建ConfigMap数据的目录，该目录中所有文件名称会作为ConfigMap中的key，文件中的内容作为value添加到新创建的ConfigMap中。

查看新创建的ConfigMap:

```
$ kubectl describe configmaps game-config
Name:           game-config
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        121 bytes
ui.properties:          83 bytes
```

在该MapConfig的describe信息中，我们能够看到名称为上边命令指定的game-config，在Data字段能够看到两个新添加的key：game.properties和ui.properties。由于value的size较大，此处只显示了大小，要想查看value的完整内容，可以通过`kubectl get`命令：

```
$ kubectl get configmaps game-config -o yaml
```

输出结果：

```
apiVersion: v1
data:
  game.properties: |-
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:34:05Z
  name: game-config
  namespace: default
  resourceVersion: "407"-
  selfLink: /api/v1/namespaces/default/configmaps/game-config
  uid: 30944725-d66e-11e5-8cd0-68f728db1985
```

### 文件创建

通过文件创建ConfigMap的命令与通过目录创建的命令相同，但--from-file参数的值为指定的文件，并且可以通过添加多个--from-file参数，同时添加多个文件。下面的命令能够得到上面命令相同的结果。

```
$ kubectl create configmap game-config-2 --from-file=docs/user-guide/configmap/kubectl/game.properties --from-file=docs/user-guide/configmap/kubectl/ui.properties
```

查看ConfigMap：

```
$ kubectl get configmaps game-config-2 -o yaml
```

```
apiVersion: v1
data:
  game.properties: |-
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:52:05Z
  name: game-config-2
  namespace: default
  resourceVersion: "516"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-2
  uid: b4952dc3-d670-11e5-8cd0-68f728db1985
```

在上面的示例中ConfigMap的key名称默认都采用的文件的名称，也可以通过`key=value`的方式指定key的名称。

```
$ kubectl create configmap game-config-3 --from-file=game-special-key=docs/user-guide/configmap/kubectl/game.properties
```

```
$ kubectl get configmaps game-config-3 -o yaml
```

```
apiVersion: v1
data:
  game-special-key: |-
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:54:22Z
  name: game-config-3
  namespace: default
  resourceVersion: "530"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-3
  uid: 05f8da22-d671-11e5-8cd0-68f728db1985
```

### key-value值创建

可以通过在命令行中直接指定ConfigMap中key、value的方式创建新的ConfigMap。参数为`--from-literal`，值为`key=value`格式的键值对，支持同时添加多个`--from-literal`参数。

```
$ kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
```

查看结果：

```
$ kubectl get configmaps special-config -o yaml
```

```
apiVersion: v1
data:
  special.how: very
  special.type: charm
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: special-config
  namespace: default
  resourceVersion: "651"
  selfLink: /api/v1/namespaces/default/configmaps/special-config
  uid: dadce046-d673-11e5-8cd0-68f728db1985
```

## ConfigMap在Pod中的使用方式

### 通过ConfigMap设置环境变量

ConfigMap中的值可以通过环境变量的方式在容器中使用，参考如下示例：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

在容器中引用该ConfigMap的值

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
  restartPolicy: Never
```

当该容器运行时，输出环境变量的值为：

```
SPECIAL_LEVEL_KEY=very
SPECIAL_TYPE_KEY=charm
```

### 通过ConfigMap设置命令行参数

ConfigMap也可以通过kubernetes替换语法`$(VAR_NAME)`来设置容器中的命令或参数的值。如下示例：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

首先需要在容器的环境变量中引入上边ConfigMap的值，然后通过`$(VAR_NAME)`语法在命令中引用它们。配置如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
  restartPolicy: Never
```

当该容器运行时，命令行的输出结果为：

```
very charm
```

### 通过volume插件使用ConfigMap

可以通过volume的方式使用ConfigMap中的值，依然采用上面的ConfigMap示例：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

通过volume的方式引用ConfigMap的值有多种方式，最基本的方法为文件方式，volume中的文件名称为ConfigMap中的key，文件内容为ConfigMap中的value。引用示例如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "cat /etc/config/special.how" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
  restartPolicy: Never
```

该容器运行时的输出结果为：

```
very
```

设置ConfigMap中key的映射路径：

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "cat /etc/config/path/to/special-key" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: path/to/special-key
  restartPolicy: Never
```

该容器运行时的输出结果为：

```
very
```

## 实战演练：配置Redis

下面通过Redis来展示ConfigMap在实际应用中的使用方法。假设我们需要将以下配置作为redis的运行参数：

```
maxmemory 2mb
maxmemory-policy allkeys-lru
```

该配置的文件目录为：`docs/user-guide/configmap/redis/redis-config`，创建ConfigMap：

```
$ kubectl create configmap example-redis-config --from-file=docs/user-guide/configmap/redis/redis-config
```

查看该ConfigMap：

```
$ kubectl get configmap example-redis-config -o yaml
```

```
apiVersion: v1
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru
kind: ConfigMap
metadata:
  creationTimestamp: 2016-03-30T18:14:41Z
  name: example-redis-config
  namespace: default
  resourceVersion: "24686"
  selfLink: /api/v1/namespaces/default/configmaps/example-redis-config
  uid: 460a2b6e-f6a3-11e5-8ae5-42010af00002
```

Redis的配置文件：

```
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: kubernetes/redis:v1
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config
          path: redis.conf
```

该配置中通过volume的方式使用ConfigMap，将ConfigMap：example-redis-config中key：redis-config的值，写入到文件redis.conf中。然后将该文件挂载到目录/redis-master下，redis在启动时会加载该文件。

创建Redis容器：

```
$ kubectl create -f docs/user-guide/configmap/redis/redis-pod.yaml
```

使用`kubectl exec`命令进入该容器，并使用`redis-cli`工具检查配置是否正确

```
$ kubectl exec -it redis redis-cli
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "2097152"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"
```

## 使用限制



