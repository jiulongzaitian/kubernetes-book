# Docker Compose搭建mysql主从复制

系统环境

* docker 1.12.3
* mysql5.7.17

# 要点说明 {#section-1}

* 使用`docker bridge`网络,设置静态IP
* 使用`volumes`挂载,不使用数据卷容器
* 镜像使用`build`创建\(保留扩展性\),不使用`image`
* 目前为止,没有暴露端口号,只是两个`slave`link了`master`.马上着手研究使用[mycat](http://www.mycat.org.cn/)完成mysql的主从复制+读写分离,敬请期待.
* 挂载`hosts`文件,以便于使用`hostname`代替ip地址

# Begin {#begin}

### `docker-engine`安装 {#docker-engine}

这个直接参考官方文档吧.[debian下安装docker-engine](https://docs.docker.com/engine/installation/linux/debian/)

### `docker-compose`安装 {#docker-compose}

[debian下安装docker-compose](https://docs.docker.com/compose/install/)

### 拉取`mysql:5.7.17`镜像 {#mysql5717}

`docker pull mysql:5.7.17`

### 需要挂载的配置文件 {#section-2}

##### 简要说明 {#section-4}

* mysql-master: 存放master配置文件
* mysql-s1: 存放第一个slave配置文件
* mysql-s2: 存放第二个slave配置文件
* hosts: 本地路由

#### mysql-master的配置 {#mysql-master}

没多少东西,只有一个`mysqld.cnf`需要在末尾追加:

```
  #表名不区分大小写
  lower_case_table_names=1
  #给数据库服务的唯一标识，一般为大家设置服务器Ip的末尾号
  server-id=2
  log-bin=master-bin
  log-bin-index=master-bin.index
```

#### mysql-s1/mysql-s2的配置 {#mysql-s1mysql-s2}

跟master一样,也只有一个`mysqld.cnf`需要在末尾追加:

```
  server-id=3 #第一个用3,第二个用4
  log-bin=s1-bin.log #第一个用s1-bin.log,第二个用s2-bin.log
  sync_binlog=1
  lower_case_table_names=1
```

#### hosts文件配置 {#hosts}

```
127.0.0.1    localhost
172.18.0.2    m1
172.18.0.3    s1
172.18.0.4    s2
```

### docker-compose配置文件和Dockerfile {#docker-composedockerfile}

##### Dockerfile {#section-5}

其实`master s1 s2`的Dockerfile都是一致的,咱就是为了保持一定的扩展性才这么写的. 我们完全可以用`docker-compose`的`image`代替.

**在docker-compose.yml中image和build不能一起使用的**

好吧,看一下这个`Dockerfile`

```
FROM mysql:5.7.17
MAINTAINER 
<
ssab work_wjj@163.com
>

EXPOSE 3306
CMD ["mysqld"]
```

##### docker-compose.yml {#docker-composeyml}

好吧,重点来了.

    version: '2' # 这个version是指dockerfile解析时用的版本,不是给我们自己定义版本号用的.
    services:
      m1: # master
        build: ./master # ./master文件下需要有Dockerfile文件,并且build属性和image属性不能一起使用
        container_name: m1 # 容器名
        volumes: # 挂载 下边每行前边的`-`代表这个东西是数组的一个元素.就是说volumes属性的值是一个数组
          - /home/ssab/config/mysql-master/:/etc/mysql/:ro # 注意改下映射关系
          - /etc/localtime:/etc/localtime:ro
          - /home/ssab/config/hosts:/etc/hosts:ro # 注意改下映射关系
        networks: # 网络
          mysql: # 见跟services平级的networks,在最下边
            ipv4_address: 172.18.0.2 # 设置静态ipv4的地址
        ulimits: # 操作系统限制
          nproc: 65535
        hostname: m1 # hostname
        mem_limit: 1024m # 最大内存使用不超过1024m,我在本地机器上测试,才只写了1024m,生产上需要根据自己的服务器配置,以及docker容器数进行调优.
        restart: always # 容器重启策略
        environment: # 设置环境变量
          MYSQL_ROOT_PASSWORD: m1test
      s1: # slave1
          build: ./s1
          container_name: s1
          volumes:
            - /home/ssab/config/mysql-s1/:/etc/mysql/:ro
            - /etc/localtime:/etc/localtime:ro
            - /home/ssab/config/hosts:/etc/hosts:ro
          networks:
            mysql:
              ipv4_address: 172.18.0.3
          links:
            - m1
          ulimits:
            nproc: 65535
          hostname: s1
          mem_limit: 1024m
          restart: always
          environment:
            MYSQL_ROOT_PASSWORD: s1test
      s2:# slave2
        build: ./s2
        container_name: s2
        volumes:
          - /home/ssab/config/mysql-s2/:/etc/mysql/:ro
          - /etc/localtime:/etc/localtime:ro
          - /home/ssab/config/hosts:/etc/hosts:ro
        links:
          - m1
        networks:
          mysql:
            ipv4_address: 172.18.0.4
        ulimits:
          nproc: 65535
        hostname: s2
        mem_limit: 1024m
        restart: always
        environment:
          MYSQL_ROOT_PASSWORD: s2test
    networks: # docker网络设置
      mysql: # 自定义网络名称
        driver: bridge # 桥接
        ipam: # 要使用静态ip必须使用ipam插件
          driver: default
          config:
          - subnet: 172.18.0.0/24
            gateway: 172.18.0.1

### run {#run}

在`docker-compose.yml`文件的目录下运行

```
  docker-compose up -d
```

别激动,我们现在才只是完成了一半….

### 设置mysql主从复制. {#mysql}

#### 配置master {#master}

* 进入master的mysql命令行

  ```
  docker exec -it m1 /bin/bash
  ```

  ```
  mysql -u root -p
  ```

  输入`MYSQL_ROOT_PASSWORD:`的值m1test,进入mysql命令行模式.

* 创建用于主从复制的用户`repl`

  ```
  mysql
  >
   create user repl;
  ```

* 给`repl`用户授予slave的权限

  ```
  #repl用户必须具有REPLICATION SLAVE权限，除此之外没有必要添加不必要的权限，密码为repl。说明一下172.18.0.%，这个配置是指明repl用户所在服务器，这里%是通配符，表示172.18.0.0-172.18.0.255的Server都可以以repl用户登陆主服务器。当然你也可以指定固定Ip。
  mysql
  >
   GRANT REPLICATION SLAVE ON *.* TO 'repl'@'172.18.0.%' IDENTIFIED BY 'repl';
  ```

* 锁库,不让数据再进行写入动作,这个命令在结束终端会话的时候会自动解锁

  ```
  FLUSH TABLES WITH READ LOCK;
  ```

* 查看master状态

  ```
  mysql
  >
   show master status;
  ```

  显示如下:

  ```
  +-------------------+----------+--------------+------------------+-------------------+
  | File              | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
  +-------------------+----------+--------------+------------------+-------------------+
  | master-bin.000003 |      636 |              |                  |                   |
  +-------------------+----------+--------------+------------------+-------------------+
  1 row in set (0.00 sec)
  ```

  记下`master-bin.000003`和`636`一会在slave用.

#### 配置slave1 {#slave1}

* 进入s1的mysql命令行

  ```
  docker exec -it s1 /bin/bash
  ```

  ```
  mysql -u root -p
  ```

输入`MYSQL_ROOT_PASSWORD:`的值s1test,进入mysql命令行模式.

* 连接master

  ```
  mysql
  >
   change master to master_host='m1',master_port=3306,master_user='repl',master_password='repl',master_log_file='master-bin.000003',master_log_pos=636;
  ```

* 启动slave

  ```
  mysql
  >
   start salve;
  ```

#### 配置slave2 {#slave2}

几乎跟slave一致….咱就不写了…

### 实验 {#section-6}

好了,到此位置,配置已经完成,那是否成功了捏… 我们来试一下.

#### 测试master写入后是否能够同步到slave {#masterslave}

* 在master的mysql命令行下创建数据库:`ms-test`

  ```
  mysql
  >
   create database mstest;
  ```

* 去两台slave上查看是否也有了mstest数据库.

  ```
  mysql
  >
   show databases;
  ```

  如果显示如下:

  ```
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mstest             |
  | mysql              |
  | performance_schema |
  | sys                |
  +--------------------+
  5 rows in set (0.00 sec)
  ```

  则证明成功.

#### 总结 {#section-7}

通过以上步骤,咱们搭建了一个以`docker-compose`管理的mysql`master-slave`模式的主从复制. 下一步,我们需要进行暴露端口,或者使用`links`属性来让应用或者其他客户端工具能够访问我们的mysql. 再下一步,我们需要使用`mycat`中间件来完成我们的读写分离.

