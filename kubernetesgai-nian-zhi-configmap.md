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



