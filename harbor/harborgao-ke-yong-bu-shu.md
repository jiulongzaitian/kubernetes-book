一个Harbor实例就是一组由[docker-compose](https://github.com/docker/compose)工具启动的容器服务，主要包括四个主要组件：

* proxy  
  实质就是一个反向代理[nginx](http://tonybai.com/tag/nginx)，负责流量路由分担到ui和registry上；

* registry  
  这里的registry就是原生的docker官方的registry镜像仓库，Harbor在内部内置了一个仓库，所有仓库的核心功能均是由registry完成的；

* core service  
  包含了ui、token和webhook服务；

* job service  
  主要用于镜像复制供。

同时，每个Harbor实例还启动了一个MySQL数据库容器，用于保存自身的配置和镜像管理相关的关系数据。

高可用系统一般考虑三方面：计算高可用、存储高可用和网络高可用。在这里我们不考虑网络高可用。基于Harbor的高可用仓库方案，这里列出两个。

![](http://tonybai.com/wp-content/uploads/harbor-ha-solutions.png "img{512x368}")

两个方案的共同点是计算高可用，都是通过lb实现的多主热运行，保证无单点；存储高可用则各有各的方案。一个使用了分布式共享存储，数据可靠性由共享存储provider提供；另外一个则需要harbor自身逻辑参与，通过镜像相互复制的方式保持数据的多副本。

两种方案各有优缺点，就看哪种更适合你的组织以及你手里的资源是否能满足方案的搭建要求。

方案1是Harbor开发团队推荐的标准方案，由于基于分布式共享存储，因此其scaling非常好；同样，由于多Harbor实例共享存储，因此可以保持数据是实时一致的。方案1的不足也是很明显的，第一：门槛高，需要具备共享存储provider；第二搭建难度要高于第二个基于镜像复制的方案。

方案2的优点就是首次搭建简单。不足也很多：scaling差，甚至是不能，一旦有三个或三个以上节点，可能就会出现“环形复制”；镜像复制需要时间，因此存在多节点上数据周期性不一致的情况；Harbor的镜像复制规则以Project为单位配置，因此一旦新增Project，需要在每个节点上手工维护复制规则，非常繁琐。因此，我们选择方案1。

我们来看一下方案1的细节： 这是一幅示意图。

* 每个安放harbor实例的node都mount cephfs。ceph是目前最流行的分布式共享存储方案之一；
* 每个node上的harbor实例（包含组件：ui、registry等）都volume mount node上的cephfs mount路径；
* 通过Load Balance将request流量负载到各个harbor实例上；
* 使用外部MySQL cluster替代每个Harbor实例内部自维护的那个MySQL容器；对于MySQL cluster，可以使用
  [mysql galera cluster](http://galeracluster.com/products/)
  或MySQL5.7以上版本自带的Group Replication \(MGR\) 集群。
* 通过外部Redis实现访问Harbor ui的session共享，这个功能是Harbor UI底层MVC框架-
  [beego](https://github.com/astaxie/beego)
  提供的。

接下来，我们就来看具体的部署步骤和细节。

环境和先决条件：

* 三台VM\(Ubuntu 16.04及以上版本\)；
* [CephFS](http://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/)
  、MySQL、Redis已就绪；
* Harbor v1.1.0及以上版本；
* 一个域名：hub.tonybai.com:8070。我们通过该域名和服务端口访问Harbor，我们可以通过dns解析多ip轮询实现最简单的Load balance，虽然不完美。

### 第一步：挂载cephfs

每个安装Harbor instance的节点都要mount cephfs的相关路径，步骤包括：

```
#安装cephfs内核驱动
apt install ceph-fs-common

# 修改/etc/fstab，添加挂载指令，保证节点重启依旧可以自动挂载cephfs
xx.xx.xx.xx:6789:/apps/harbor /mnt/cephfs/harbor ceph name=harbor,secretfile=/etc/ceph/a dmin.secret,noatime,_netdev 0 2

```

这里涉及一个密钥文件admin.secret，这个secret文件可以在ceph集群机器上使用ceph auth tool生成。

![](http://tonybai.com/wp-content/uploads/harbor-prepare-process.png "img{512x368}")

前面提到过每个Harbor实例都是一组容器服务，这组容器启动所需的配置文件是在Harbor正式启动前由prepare脚本生成的，Prepare脚本生成过程的输入包括：harbor.cfg、docker-compose.yml和common/templates下的配置模板文件。这也是部署高可用Harbor的核心步骤，我们逐一来看。

### 第二步：修改harbor.cfg

我们使用域名访问Harbor，因此我们需要修改hostname配置项。注意如果要用域名访问，这里一定填写域名，否则如果这里使用的是Harbor node的IP，那么在后续会存在client端和server端仓库地址不一致的情况；

custom\_crt=false 关闭 crt生成功能。注意：三个node关闭其中两个，留一个生成一套数字证书和私钥。

### 第三步：修改docker-compose.yml

docker-compose.yml是docker-compose工具标准配置文件，用于配置docker-compose即将启动的容器服务。针对该配置文件，我们主要做三点修改：

* 修改volumes路径
 
  由/data/xxx 改为：/mnt/cephfs/harbor/data/xxx
* 由于使用外部Mysql，因此需要删除mysql service以及其他 service对mysql service的依赖 \(depends\_on\)
* 修改对proxy外服务端口 ports: 8070:80

### 第四步：配置访问external mysql和redis

external mysql的配置在common/templates/adminserver/env中，我们用external Mysql的访问方式覆盖下面四项配置：

```
MYSQL_HOST=harbor_host
MYSQL_PORT=3306
MYSQL_USR=harbor
MYSQL_PWD=harbor_password


```

还有一个关键配置，那就是将RESET由false改为true。[只有改为true，adminserver启动时，才能读取更新后的配置](http://tonybai.com/2017/06/09/setup-a-high-availability-private-registry-based-on-harbor-and-cephfs/)：

```
RESET=true

```

Redis连接的配置在common/templates/ui/env中，我们需要新增一行：

```
_REDIS_URL=redis_ip:6379,100,password,0

```

### 第五步：prepare并启动harbor

执行prepare脚本生成harbor各容器服务的配置；在每个Harbor node上通过下面命令启动harbor实例：

```
docker-compose up -d


```

启动后，可以通过docker-compose ps命令查看harbor实例中各容器的启动状态。如果启动顺利，都是”Up”状态，那么我们可以在浏览器里输入：http://hub.tonybai.com:8070，不出意外的话，我们就可以看到Harbor ui的登录页面了。

至此，我们的高可用Harbor cluster搭建过程就告一段落了。

  


