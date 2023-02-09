
# Docker搭建过程参考

- [写在前面](#写在前面)
- [搭建过程](#搭建过程)
    - [0. Registry&UI](#0-registryui)
    - [1. MySQL](#1-mysql)
    - [2. Redis](#2-redis)
    - [3. Rabbitmq](#3-rabbitmq)
    - [4. Mongo](#4-mongo)
    - [5. PostgreSQL](#5-postgresql)
    - [6. SonarQube](#6-sonarqube)
    - [7. Nginx](#7-nginx)
    - [8. Nacos](#8-nacos)
    - [9. Sentinel-Dashboard](#9-sentinel-dashboard)
    - [10.FTP](#10-ftp)
    - [11.Jenkins](#11-jenkins)
    - [12.SqlServer](#12-sqlserver)
    - [13.Oracle](#13-oracle)
    - [14.Seata Server](#14-seata-server)
    - [15.Nexus](#15-nexus)
    - [16.Grafana](#16-grafana)
    - [17.Zookeeper Cluster](#17-zookeeper-cluster)
    - [18.Kafka Cluster](#18-kafka-cluster)
    - [19.Elasticsearch Cluster](#19-elasticsearch-cluster)
    - [20.Kibana](#20-kibana)
    - [21.HBase](#21-hbase)
    - [22.GitLab](#22-gitlab)
    - [23.SVN](#23-svn)
    - [24.禅道](#24-zentao)
    - [25.ZipKin](#25-zipkin)
- [部分命令解释](#部分命令解释)

----------------------------------------------------

# 写在前面

**安装Docker**

* 安装最新版本的 Docker Engine-Community 和 containerd
  ```shell
  sudo yum install docker-ce docker-ce-cli containerd.io
  ```
* 或列出并排序存储库中可用的版本。此示例按版本号（从高到低）对结果进行排序
  ```shell
  yum list docker-ce --showduplicates | sort -r
  ```
  > docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable  
  docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable  
  docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable  
  docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable

  通过其完整的软件包名称安装特定版本，该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）一直到第一个连字符，并用连字符（-）分隔。例如：docker-ce-18.09.1
  ```shell
  sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
  ```

**启动docker**

* 启动
  ```shell
  sudo systemctl enable docker ## 自启动
  sudo systemctl start docker
  ```
* 验证
  ```shell
  sudo docker run hello-world
  ```

**卸载docker**

* 删除安装包
  ```shell
  yum remove docker-ce
  ```
* 删除镜像、容器、配置文件等内容
  ```shell
  rm -rf /var/lib/docker
  ```

**超时问题**(目前看来，好像不是那么容易超时了。*since 2021.05.06*)

由于 https://hub.docker.com/ 非常容易超时，所以需要改一下docker镜像源  
找到/etc/docker/daemon.json，添加

```shell
{
  "registry-mirrors" :["https://docker.mirrors.ustc.edu.cn"]
}
```

> 个人镜像加速器地址：https://xvaz3vtq.mirror.aliyuncs.com

**如果高版本docker出现问题，可以使用其他方式**

```shell 
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud   
```

**docker0 IP地址修改方法**

```shell
vi /etc/docker/daemon.json
```

增加内容

```shell
{  
  "registry-mirrors": ["https://xvaz3vtq.mirror.aliyuncs.com"] ,  
  "bip":"172.17.0.1/16"  
}
```

> docker间互相访问时，可以使用172.17.0.1，而非localhost

**调整时区**

```shell 
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

或

```shell 
docker cp /etc/localtime [容器ID或者NAME]:/etc/localtime
```

**权限问题**
1. 粗暴的解决方案
  ```shell
  chmod 777 [文件夹]
  ```
2. 或者在启动docker容器时，增加`--privileged=true`命令

**java项目时区问题**

```shell
env JAVA_OPTS="$JAVA_OPTS -Duser.timezone=GMT+08"
```

**安装docker-compose**

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

```shell
sudo chmod +x /usr/local/bin/docker-compose
```

**卸载docker-compose**

```shell
sudo rm /usr/local/bin/docker-compose
```

----------------------------------------------------

# 搭建过程

## 0. Registry&UI

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/repository/registry
```

* 执行
    - 私有仓库

    ```shell
    docker pull registry:2.7.1
    ```

    ```shell
    docker run -d -v [文件存放目录]/repository/registry:/var/lib/registry \
    -p 15000:5000 \
    --restart=always \ 
    --name registry registry:2.7.1
    ```

  > 首先，为了让客户端服务器能够快速地访问刚刚在服务端搭建的镜像仓库(默认情况下是需要配置HTTPS证书的),  
  这里简单在客户端配置一下私有仓库的可信任设置，让我们可以通过HTTP直接访问:  
  > 执行`vim /etc/docker/daemon.json`，打开配置文件  
  加上下面这一句:
  > ```shell
  >   {
  >     "insecure-registries" : ["your-server-ip:15000"]
  >   }
  >   ```
  > 这里的"your-server-ip"请换为你的服务器的外网IP地址
  >> PS: 如果不设置可信任源，又没有配置HTTPS证书，那么会遇到这个错误：error: Get https://ip:port/v1/_ping: http: server gave HTTP response to HTTPS client.
  >
  > 为了使得配置生效，需要重新启动docker服务: `systemctl restart docker`  
  其次，为要上传的镜像打Tag:
  `docker tag your-image-name:tagname your-server-ip:15000/your-image-name:tagname`  
  最后，开始正式上传镜像到服务端镜像仓库:
  `docker push your-registry-server-ip:15000/your-image-name:tagname`  
  例:
  > ```shell
  >   docker tag auth:latest localhost:15000/auth:1.0
  >   docker push localhost:15000/auth:1.0
  > ```

    - UI界面

      ```shell
      docker pull hyper/docker-registry-web
      ```

      ```shell
      ##命令注释
      docker run                                 ##运行
      -d                                         ##后台运行
      -p 15001:8080                              ##端口映射
      --name registry-web                        ##容器命名
      --link registry                            ##连接其他容器  加入registry到host
      -e REGISTRY_URL=http://172.17.0.1:15000/v2 ##指定仓库地址
      -e REGISTRY_NAME=localhost:15000           ##仓库命名
      hyper/docker-registry-web                  ##被启动的镜像
      ```

* 或者使用docker compose，将两者绑定到一起

  文件内容如下：

  ```yaml
  version: "2"
  services:
    registry:
      image: registry:2.7.1
      container_name: registry
      volumes:
        - [文件存放目录]/repository/registry:/var/lib/registry
      ports:
        - "15000:5000"
    registry-web:
      image: hyper/docker-registry-web:latest
      container_name: registry-web
      environment:
        - REGISTRY_URL=http://172.17.0.1:15000/v2
        - REGISTRY_NAME=localhost:15000
      ports:
        - "15001:8080"
      depends_on:
        - registry
  ```
  - 执行：
  ```shell
  docker-compose -f docker-compose.yaml -p repository up -d
  ```


* [其他参考](https://docs.docker.com/registry/)

## 1. MySQL

### 单机模式



* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/mysql/8.0.31/data
mkdir -p [文件存放目录]/mysql/8.0.31/conf
```

[文件存放目录]/mysql/8.0.31/conf/[docker.cnf](conf/mysql/docker.cnf)  
[文件存放目录]/mysql/8.0.31/conf/[mysql.cnf](conf/mysql/mysql.cnf)  
[文件存放目录]/mysql/8.0.31/conf/[mysqldump.cnf](conf/mysql/mysqldump.cnf)  

* 执行

```shell
docker pull mysql:8.0.31
```

```shell
docker run -p 13306:3306 --name mysql8.0.31 \
-v [文件存放目录]/mysql/8.0.31/data:/var/lib/mysql \
-v [文件存放目录]/mysql/8.0.31/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD='passw0rd' -d mysql:8.0.31
```

```shell
mysql -u root -p
'passw0rd'
```

```mysql
CREATE USER 'carzer'@'%' IDENTIFIED WITH mysql_native_password BY 'passw0rd';
GRANT ALL PRIVILEGES ON *.* TO 'dbname'@'%';
flush privileges;
```

### 主从模式

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/mysql-cluster/master/data
mkdir -p [文件存放目录]/mysql-cluster/master/conf
mkdir -p [文件存放目录]/mysql-cluster/slave/data
mkdir -p [文件存放目录]/mysql-cluster/slave/conf
```
  - 主  
    [文件存放目录]/mysql-cluster/master/[conf](./conf/mysql/master)/
  - 从  
    [文件存放目录]/mysql-cluster/slave/[conf](./conf/mysql/slave)/  

* 创建网络
  `docker network create mysql-cluster --driver bridge`

* 创建docker-compose.yaml  
```yaml
version: "3.9"

networks:
  mysql-cluster:
    external: true
    name: mysql-cluster

services:
  master:
    image: mysql:5.7.29
    hostname: mysql-master
    container_name: mysql-master
    networks:
      - mysql-cluster
    ports:
      - '13306:3306'
    volumes:
      - [文件存放目录]/mysql-cluster/master/data:/var/lib/mysql
      - [文件存放目录]/mysql-cluster/master/conf:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: 'passw0rd'
      MYSQL_DATABASE: 'dbname'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'passw0rd'
  slave:
    image: mysql:5.7.29
    hostname: mysql-slave
    container_name: mysql-slave
    networks:
      - mysql-cluster
    ports:
      - '13360:3306'
    volumes:
      - [文件存放目录]/mysql-cluster/slave/data:/var/lib/mysql
      - [文件存放目录]/mysql-cluster/slave/conf:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: 'passw0rd'
      MYSQL_DATABASE: 'dbname'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'passw0rd'
    depends_on:
      - master
```

* 执行
```shell
docker-compose -f docker-compose.yaml -p mysql-cluster up -d
```

* 建立主从连接

连接master数据库，创建slave用户：
```shell
CREATE USER 'slave'@'%' IDENTIFIED BY 'slave密码';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
```
查看master状态：
`show master status;`

| File                 | Position | Bin log_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
|----------------------|----------|---------------|------------------|-------------------|
| edu-mysql-bin.000003 | 154      |               | mysql            |                   |

然后连接slave数据库，执行连接命令：
```shell
# 这里的'master'是主库的hostname，实际部署时，需要使用IP或者配置hosts
# 然后根据master的信息，配置'File/master_log_file'、'Position/master_log_pos'信息
change master to master_host='master', master_port=3306, master_log_file='edu-mysql-bin.000003', master_log_pos=154, master_connect_retry=30;  
```
然后执行：
```shell
start slave USER='slave' PASSWORD='slave密码';
```
查看slave状态：
`show slave status ;`
看到`SlaveIORunning`和`SlaveSQLRunning`是`Yes`表明已经开始工作了，如果是`No`，则可以重启后尝试。如果依然无法运行，可查看字段`Last_Error`中的信息进行处理。

其他可能用到的命令：
```shell
stop slave; # 停止从服务
reset slave all; # 重置所有从服务信息
```


## 2. Redis

### 单机模式

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/redis/conf
mkdir -p [文件存放目录]/redis/data
mkdir -p [文件存放目录]/redis/data/log
```

[文件存放目录]/redis/conf/[redis.conf](conf/redis/redis.conf)

* 执行

```shell
docker pull redis:6.2.6
```

```shell
docker run -d --privileged=true -p 6379:6379 \
-v [文件存放目录]/redis/conf/redis.conf:/etc/redis/redis.conf \
-v [文件存放目录]/redis/data:/data \
--name redis redis:6.2.6 redis-server /etc/redis/redis.conf \
--appendonly yes
```

### 哨兵集群
> compose文件及相关配置都在 [redis-cluster](conf/redis/redis-cluster) 中  
sentinel相关的配置文件，为了方便本地使用，监听了本地修改后的host：carzer.com，可以根据实际情况自行修改IP

## 3. Rabbitmq

### 单机模式
* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/rabbitmq/data
```

* 执行

```shell
docker pull rabbitmq:3.9.11-management-alpine
```

```shell
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 \
-v [文件存放目录]/rabbitmq/data:/var/lib/rabbitmq \
--hostname myRabbit \
-e RABBITMQ_DEFAULT_VHOST=my_vhost \
-e RABBITMQ_DEFAULT_USER=admin \
-e RABBITMQ_DEFAULT_PASS=admin rabbitmq:3.9.11-management-alpine
```

### 集群模式

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/rabbitmq-cluster/conf

mkdir -p [文件存放目录]/rabbitmq-cluster/rabbit1/data
mkdir -p [文件存放目录]/rabbitmq-cluster/rabbit2/data
mkdir -p [文件存放目录]/rabbitmq-cluster/rabbit3/data
```

[文件存放目录]/rabbitmq-cluster/conf/[rabbitmq-cluster.conf](./conf/rabbitmq/rabbitmq-cluster.conf)，密码在其中进行配置  

* 执行

```shell
docker pull rabbitmq:3.9.11-management-alpine
```
* 创建网络
  `docker network create rabbit-cluster --driver bridge`

* 创建docker-compose.yaml

```yaml
version: "3.9"

networks:
  rabbit-cluster:
    external: true
    name: rabbit-cluster

services:
  rabbit1:
    image: rabbitmq:3.9.11-management-alpine
    hostname: rabbit1
    container_name: rabbit1
    networks:
      - rabbit-cluster
    ports:
      - '5673:5672'
      - '15673:15672'
    volumes:
      - [文件存放目录]/rabbitmq-cluster/rabbit1/data:/var/lib/rabbitmq
      - [文件存放目录]/rabbitmq-cluster/conf/rabbitmq-cluster.conf:/etc/rabbitmq/rabbitmq.conf
    environment:
      RABBITMQ_ERLANG_COOKIE: rabbit_cluster_cookie
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.hostname==node1
  rabbit2:
    image: rabbitmq:3.9.11-management-alpine
    hostname: rabbit2
    container_name: rabbit2
    networks:
      - rabbit-cluster
    ports:
      - '5674:5672'
      - '15674:15672'
    volumes:
      - [文件存放目录]/rabbitmq-cluster/rabbit2/data:/var/lib/rabbitmq
      - [文件存放目录]/rabbitmq-cluster/conf/rabbitmq-cluster.conf:/etc/rabbitmq/rabbitmq.conf
    environment:
      RABBITMQ_ERLANG_COOKIE: rabbit_cluster_cookie
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.hostname==node2
  rabbit3:
    image: rabbitmq:3.9.11-management-alpine
    hostname: rabbit3
    container_name: rabbit3
    networks:
      - rabbit-cluster
    ports:
      - '5675:5672'
      - '15675:15672'
    volumes:
      - [文件存放目录]/rabbitmq-cluster/rabbit3/data:/var/lib/rabbitmq
      - [文件存放目录]/rabbitmq-cluster/conf/rabbitmq-cluster.conf:/etc/rabbitmq/rabbitmq.conf
    environment:
      RABBITMQ_ERLANG_COOKIE: rabbit_cluster_cookie
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.hostname==node3
```
* 执行：  
```shell
docker-compose -f docker-compose.yaml -p rabbit-cluster up -d
```

* 查看集群状态：
```shell
docker exec -it rabbit1 rabbitmqctl cluster_status
```

## 4. Mongo

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/mongo/5.0.5/data
```

* 执行

```shell
docker pull mongo:5.0.5
```

```shell
docker run --name mongo5.0.5 \
-p 27017:27017 \
-v [文件存放目录]/mongo/5.0.5/data/db:/data/db \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD='admin' \
-d mongo:5.0.5 --auth
```

在MacOS或Windows中，由于文件结构问题，需要先为mongodb创建挂载卷`docker volume create mongo-vol`，  
然后使用挂载卷来持久化数据`-v [文件存放目录]/mongo/data/db:/data/db`变更为`-v mongo-vol:/data/db`  

* 可能用到的命令
```shell
use admin
db.createUser({ user:'admin',pwd:'密码',roles:[{role:'userAdminAnyDatabase',db:'admin'}]})
db.auth("admin","密码")
db.grantRolesToUser("admin",[{role:"dbOwner", db:"admin"}])
```
* 改密码
```shell
db.changeUserPassword('admin','新密码');  
```

## 5. PostgreSQL

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/postgres/data
```

* 执行

```shell
docker pull postgres:13.5-bullseye
```

```shell
docker run -d --name postgres -p 5432:5432 \
-v [文件存放目录]/postgresql/data:/var/lib/postgresql/data \
-e POSTGRES_USER=sonar \
-e POSTGRES_PASSWORD=sonar \
--privileged=true \
postgres:13.5-bullseye
```

## 6. SonarQube

> 当前使用版本为9.2-community  
sonarqube7.9以后，不再对MySQL提供支持，这里使用了PostgreSQL

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/sonarqube/logs
mkdir -p [文件存放目录]/sonarqube/data
mkdir -p [文件存放目录]/sonarqube/extensions
```

* 执行

```shell
docker pull sonarqube:9.2-community
```

```shell
docker run -d --name sonar -p 9000:9000 -p 9092:9092 \
-v [文件存放目录]/sonarqube/logs:/opt/sonarqube/logs \
-v [文件存放目录]/sonarqube/data:/opt/sonarqube/data \
-v [文件存放目录]/sonarqube/extensions:/opt/sonarqube/extensions \
-e SONARQUBE_JDBC_USERNAME=sonar \
-e SONARQUBE_JDBC_PASSWORD=sonar \
-e SONARQUBE_JDBC_URL=jdbc:postgresql://172.17.0.1:5432/sonar sonarqube:9.2-community
```

默认用户名密码：admin/admin

> max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
elasticsearch启动时遇到的错误

在宿主机的 /etc/sysctl.conf文件最后添加一行 `vm.max_map_count=262144`,然后执行：

```shell
sysctl -p
```

## 7. Nginx

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/nginx
mkdir -p [文件存放目录]/nginx/ext
mkdir -p [文件存放目录]/nginx/logs
mkdir -p [文件存放目录]/nginx/html
```

[文件存放目录]/nginx/[nginx.conf](conf/nginx/nginx.conf)

* 执行

```shell
docker pull nginx:1.21.4
```

```shell
docker run --name nginx -p 80:80 \
-v [文件存放目录]/nginx/nginx.conf:/etc/nginx/nginx.conf \
-v [文件存放目录]/nginx/mime.types:/etc/nginx/mime.types \
-v [文件存放目录]/nginx/ext:/etc/nginx/ext \
-v [文件存放目录]/nginx/logs:/etc/nginx/logs \
-v [文件存放目录]/nginx/html:/etc/nginx/html \
--privileged=true \
-d nginx:1.21.4
```

如果需要证书，可以追加`-v [文件存放目录]/nginx/certs:/etc/nginx/certs \`

* 备用命令
```shell
upstream bakend {
    server 192.168.1.10 weight=1 max_fails=10 fail_timeout=60s; # weight为负载权重，不配置时，默认为1
    server 192.168.1.11 weight=1 max_fails=10 fail_timeout=60s; # weight数值越大，被访问的几率越高，压力也越大
}
```
**命令解释**  
1. weight为负载权重，不配置时，默认为1。weight数值越大，被访问的几率越高，压力也越大。
2. max_fails与fail_timeout需要搭配使用，以上面的配置为例：60s内，失败10次，则停止分发请求，再过60s会重新判断服务是否可用。

## 8. Nacos

默认用户名密码：nacos/nacos  

### 单机模式

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/nacos/logs
mkdir -p [文件存放目录]/nacos/init.d
mkdir -p [文件存放目录]/nacos/data
```

[文件存放目录]/nacos/init.d/[custom.properties](./conf/nacos/custom.properties)

* 创建docker-compose.yaml

```yaml
version: "3"
services:
  nacos:
    image: nacos/nacos-server:1.4.2
    hostname: nacos
    container_name: nacos
    ports:
      - "18848:8848"
    environment:
      - PREFER_HOST_MODE=hostname
      - MODE=standalone
      - NACOS_AUTH_ENABLE=true
    volumes:
      - [文件存放目录]/nacos/logs:/home/nacos/logs
      - [文件存放目录]/nacos/init.d/custom.properties:/home/nacos/init.d/custom.properties
      - [文件存放目录]/nacos/data/:/home/nacos/data/
```

* 执行

```shell
docker-compose -f docker-compose.yaml -p nacos up -d
```

### 集群模式

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/nacos-cluster/env

mkdir -p [文件存放目录]/nacos-cluster/nacos1/logs
mkdir -p [文件存放目录]/nacos-cluster/nacos1/init.d
mkdir -p [文件存放目录]/nacos-cluster/nacos1/data

mkdir -p [文件存放目录]/nacos-cluster/nacos2/logs
mkdir -p [文件存放目录]/nacos-cluster/nacos2/init.d
mkdir -p [文件存放目录]/nacos-cluster/nacos2/data

mkdir -p [文件存放目录]/nacos-cluster/nacos3/logs
mkdir -p [文件存放目录]/nacos-cluster/nacos3/init.d
mkdir -p [文件存放目录]/nacos-cluster/nacos3/data
```
* 文件&文件夹准备  

运行时环境：[文件存放目录]/nacos-cluster/env/[nacos-cluster.env](./conf/nacos/nacos-cluster.env)  
mysql数据库初始化脚本：[nacos-mysql.sql](./conf/nacos/nacos-mysql.sql),github[地址](https://github.com/alibaba/nacos/blob/develop/distribution/conf/nacos-mysql.sql)  

[文件存放目录]/nacos-cluster/nacos1/init.d/[custom.properties](./conf/nacos/custom.properties)
[文件存放目录]/nacos-cluster/nacos2/init.d/[custom.properties](./conf/nacos/custom.properties)
[文件存放目录]/nacos-cluster/nacos3/init.d/[custom.properties](./conf/nacos/custom.properties)

* 创建网络  
  `docker network create nacos-cluster --driver bridge`  
  nacos-cluster.env文件部分参数说明:  
>在docker环境下，需要执行：`docker network connect nacos-cluster mysql`命令，才能让nacos容器识别`MYSQL_SERVICE_HOST=mysql`  
如果未将mysql`connect`，或者非当前docker环境，需要将`MYSQL_SERVICE_HOST`修改为mysql宿主机IP  
另外，别忘记修改nacos密码：`MYSQL_SERVICE_PASSWORD`

* 创建docker-compose.yaml

```yaml
version: "3.9"

networks:
  nacos-cluster:
    external: true
    name: nacos-cluster

services:
  nacos1:
    image: nacos/nacos-server:1.4.2
    hostname: nacos1
    container_name: nacos1
    networks:
      - nacos-cluster
    ports:
      - "18849:8848"
    volumes:
      - [文件存放目录]/nacos-cluster/nacos1/logs:/home/nacos/logs
      - [文件存放目录]/nacos-cluster/nacos1/init.d/custom.properties:/home/nacos/init.d/custom.properties
      - [文件存放目录]/nacos-cluster/nacos1/data/:/home/nacos/data/
    env_file:
      - [文件存放目录]/nacos-cluster/env/nacos-cluster.env
  nacos2:
    image: nacos/nacos-server:1.4.2
    hostname: nacos2
    container_name: nacos2
    networks:
      - nacos-cluster
    ports:
      - "18850:8848"
    volumes:
      - [文件存放目录]/nacos-cluster/nacos2/logs:/home/nacos/logs
      - [文件存放目录]/nacos-cluster/nacos2/init.d/custom.properties:/home/nacos/init.d/custom.properties
      - [文件存放目录]/nacos-cluster/nacos2/data/:/home/nacos/data/
    env_file:
      - [文件存放目录]/nacos-cluster/env/nacos-cluster.env
  nacos3:
    image: nacos/nacos-server:1.4.2
    hostname: nacos3
    container_name: nacos3
    networks:
      - nacos-cluster
    ports:
      - "18851:8848"
    volumes:
      - [文件存放目录]/nacos-cluster/nacos3/logs:/home/nacos/logs
      - [文件存放目录]/nacos-cluster/nacos3/init.d/custom.properties:/home/nacos/init.d/custom.properties
      - [文件存放目录]/nacos-cluster/nacos3/data/:/home/nacos/data/
    env_file:
      - [文件存放目录]/nacos-cluster/env/nacos-cluster.env
```

* 执行

```shell
docker-compose -f docker-compose.yaml -p nacos-cluster up -d
```


## 9. Sentinel-Dashboard

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/sentinel/logs
```

  - 懒人模式
    创建docker-compose.yaml  
  ```yaml
  version: "3"
  services:
    sentinel:
      container_name: sentinel
      image: bladex/sentinel-dashboard:1.8.0
      ports:
        - "8787:8858"
      volumes:
        - [文件存放目录]/sentinel/logs/:/root/logs/csp/
  ```
  默认登录用户名密码：sentinel/sentinel
  - 自行构建  
    拉取jdk8镜像`docker pull openjdk:8-jdk-slim-bullseye`  
    到[GitHub](https://github.com/alibaba/Sentinel/releases) 上下载对应版本的release包，这里选用的是[v1.8.2](https://github.com/alibaba/Sentinel/releases/download/1.8.2/sentinel-dashboard-1.8.2.jar)  
    使用[Dockerfile](./conf/sentinel/Dockerfile)自行构建[sentinel-dashboard](./conf/sentinel/sentinel-dashboard-1.8.2.jar)镜像  
    build镜像命令:`docker build --no-cache -t carzer/sentinel:1.8.2 [dockerfile所在路径]`  
    创建docker-compose.yaml  
  ```yaml
  version: "3"
  services:
    sentinel:
      container_name: sentinel
      image: carzer/sentinel:1.8.2
      ports:
        - "8787:8858"
      volumes:
        - [文件存放目录]/sentinel/logs/:/root/logs/csp/
  ```
  默认登录用户名密码：admin/admin(Dockerfile中:`-Dsentinel.dashboard.auth.username=admin`,`-Dsentinel.dashboard.auth.password=admin`)
* 执行

```shell
docker-compose -f docker-compose.yaml -p sentinel up -d
```

## 10. FTP

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/ftp/vsftpd
```

* 执行

```shell
docker pull bogem/ftp
```

`PASV_ADDRESS`需要修改为外部可访问的地址(一般是宿主机地址)

```shell
docker run -d -v [文件存放目录]/ftp/vsftpd:/home/vsftpd \
                -p 9020:20 -p 9021:21 -p 47400-47470:47400-47470 \
                -e FTP_USER=ftpuser \
                -e FTP_PASS='passw0rd' \
                -e PASV_ADDRESS=127.0.0.1 \
                --name ftp \
                --restart=always bogem/ftp
```

## 11. Jenkins

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/jenkins/sh
mkdir -p [文件存放目录]/jenkins/jenkins_home
```

[文件存放目录]/jenkins/sh/[jenkins.sh](conf/jenkins/jenkins.sh)

* 执行

```shell
docker pull jenkins/jenkins:lts-jdk11
```

```shell
docker run -d -u root -p 28080:8080 -p 50001:50001 --env JENKINS_SLAVE_AGENT_PORT=50001 \
-v /etc/localtime:/etc/localtime \
-v [文件存放目录]/jenkins/sh/jenkins.sh:/usr/local/bin/jenkins.sh \
-v [文件存放目录]/jenkins/jenkins_home:/var/jenkins_home \
--name jenkins --restart=always jenkins/jenkins:lts-jdk11
```
* 查看初始密码
```shell
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```


## 12. SqlServer

* 执行

```shell
docker pull mcr.microsoft.com/mssql/server:2019-CU3-ubuntu-18.04

docker run -e "ACCEPT_EULA=Y" -e  "SA_PASSWORD=<YourStrong@Passw0rd>" -p 14330:1433 --name sqlserver2019 \
-d mcr.microsoft.com/mssql/server:2019-CU3-ubuntu-18.04
```

由于Mac的特殊性，SqlServer在直接挂载磁盘的时候会报错：`server error 87(the parameter is incorrect.)`  
根据官网的说明：
> Important
>
> Host volume mapping for Docker on Windows does not currently support mapping the complete /var/opt/mssql directory. However, you can map a subdirectory, such as /var/opt/mssql/data to your host machine.
>
> Host volume mapping for Docker on Mac with the SQL Server on Linux image is not supported at this time. Use data volume containers instead. This restriction is specific to the /var/opt/mssql directory. Reading from a mounted directory works fine. For example, you can mount a host directory using -v on Mac and restore a backup from a .bak file that resides on the host.

[原文地址(修订时间:03/22/2021)](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-docker-container-configure?view=sql-server-ver15&pivots=cs1-bash#mount-a-host-directory-as-data-volume)

需要使用挂载卷，参照 [docker挂载卷说明](https://docs.docker.com/storage/volumes/)

```shell
docker pull mcr.microsoft.com/mssql/server:2019-CU3-ubuntu-18.04

docker volume create mssql-vol

docker run -e "ACCEPT_EULA=Y" -e  "SA_PASSWORD=<YourStrong@Passw0rd>" -p 14330:1433 \
-v mssql-vol:/var/opt/mssql --name sqlserver2019 -d mcr.microsoft.com/mssql/server:2019-CU3-ubuntu-18.04
```

```shell
docker volume inspect mssql-vol
```

> [
{
"CreatedAt": "2020-06-19T02:11:32Z",
"Driver": "local",
"Labels": {},
"Mountpoint": "/var/lib/docker/volumes/mssql-vol/_data",
"Name": "mssql-vol",
"Options": {},
"Scope": "local"
}
]

## 13. Oracle

1. Oracle12c
   * 文件&文件夹准备  

    ```shell
    mkdir -p [文件存放目录]/oracle/12c
    ```

   * 执行  

    这里选择了 [quay.io/maksymbilenko/oracle-12c](https://github.com/MaksymBilenko/docker-oracle-12c)

    ```shell
    docker pull quay.io/maksymbilenko/oracle-12c
    ```
   
    ```shell
    docker run -d -p 8088:8080 -p 1521:1521 \
    -v [文件存放目录]/oracle/12c:/u01/app/oracle \
    -e DBCA_TOTAL_MEMORY=1024 quay.io/maksymbilenko/oracle-12c
    ```
2. Oracle19c

> add since 20221209

   * 文件&文件夹准备  

    ```shell
    mkdir -p [文件存放目录]/oracle/19c
    ```  
    从github获取[oracle/docker-images](https://github.com/oracle/docker-images)，或者直接使用工程中的[OracleDatabase](./conf/OracleDatabase)  

   * 执行  

   由于选择的数据库版本为19.3.0版本创建镜像，所以建议将oracleLinux7-slim提前pull下来: `docker pull oraclelinux:7-slim`，
   然后根据[readme](./conf/OracleDatabase/SingleInstance/README.md)，将数据库文件下载并放置到合适的位置。
    
    * 参照
        - build
        ```shell
          ./buildContainerImage.sh -v 19.3.0 -t oracle/database:19.3.0 -e -i
        ```
        - run
        ```shell
          docker run --name oracle19.3.0 \
          --privileged=true -d \
          -p 19521:1521 -p 19500:5500 -p 19484:2484 \
          -e ORACLE_SID=ORCLCDB \
          -e ORACLE_PDB=ORCLPDB \
          -e ORACLE_PWD='Op123456.' \
          -e ORACLE_EDITION=enterprise \
          -e ENABLE_TCPS=true \
          -v [文件存放目录]/oracle/19.3.0/oradata:/opt/oracle/oradata \
          oracle/database:19.3.0
        ```

## 14. Seata-Server

**注意**  
seata容器暴露的端口和内部端口需要保持一致，不要有偏移量，不然注册的IP和可访问的IP会不一致，导致`connect failed, can not connect to services-server`  

### 单机模式

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/seata/config
mkdir -p [文件存放目录]/seata/logs
```

[文件存放目录]/seata/conf/[registry.conf](./conf/seata/conf/registry.conf)  
[文件存放目录]/seata/conf/[file.conf](./conf/seata/conf/file.conf)

* 执行

```shell
docker pull seataio/seata-server:1.4.2
```

* 创建docker-compose.yaml

```yaml
version: "3"
services:
  seata-server:
    image: seataio/seata-server:1.4.2
    hostname: seata-server
    container_name: seata-server
    ports:
      - "8091:8091"
    environment:
      - SEATA_IP='宿主机IP'
      - SEATA_PORT=8091
      - SEATA_CONFIG_NAME=file:/root/seata-config/registry
    volumes:
      - [文件存放目录]/seata/config:/root/seata-config
      - [文件存放目录]/seata/logs:/root/logs/seata
```

* 推送配置

```shell
cd ./conf/seata/config-center/nacos

sh nacos-config.sh 127.0.0.1
```

将[config.txt](./conf/seata/config-center/config.txt) 推送到nacos所在服务器

* 可能用到的完整命令

```shell
cd ./conf/seata/config-center/nacos
sh nacos-config.sh -h 127.0.0.1 -p 8848 -g dev -t [命名空间ID] -u nacos -w nacos
```
* 执行
```shell
docker-compose -f docker-compose.yaml -p seata-server up -d
```

### 高可用模式

在配置中心设置`store.mode=db`(当前config.txt中已配置)或`store.mode=redis`的同时，直接启动多个seata-server就可以实现高可用

这里要特别注意`SERVER_NODE`必须是每个节点唯一的，而`SEATA_ENV: cluster`会让seata在读取配置时，由原来的`registry.conf`改为读取`registry-cluster.conf`   

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/seata-cluster/config

mkdir -p [文件存放目录]/seata-cluster/logs/seata1
mkdir -p [文件存放目录]/seata-cluster/logs/seata2
mkdir -p [文件存放目录]/seata-cluster/logs/seata3
```

[文件存放目录]/seata-cluster/config/[registry-cluster.conf](./conf/seata/conf/registry-cluster.conf)

* 创建docker-compose.yaml  

```yaml
version: "3.9"

# 直接使用了已有的nacos内部网络，可根据实际情况调整
networks:
  nacos-cluster:
    external: true
    name: nacos-cluster

services:
  seata1:
    image: seataio/seata-server:1.4.2
    hostname: seata1
    container_name: seata1
    ports:
      - "18091:18091"
    networks:
      - nacos-cluster
    environment:
      SEATA_IP: '宿主机IP'
      SEATA_PORT: 18091
      SERVER_NODE: 1
      SEATA_CONFIG_NAME: file:/root/seata-config/registry
      SEATA_ENV: cluster
    volumes:
      - [文件存放目录]/seata-cluster/config:/root/seata-config
      - [文件存放目录]/seata-cluster/logs/seata1:/root/logs/seata
  seata2:
    image: seataio/seata-server:1.4.2
    hostname: seata2
    container_name: seata2
    ports:
      - "18092:18092"
    networks:
      - nacos-cluster
    environment:
      SEATA_IP: '宿主机IP'
      SEATA_PORT: 18092
      SERVER_NODE: 2
      SEATA_CONFIG_NAME: file:/root/seata-config/registry
      SEATA_ENV: cluster
    volumes:
      - [文件存放目录]/seata-cluster/config:/root/seata-config
      - [文件存放目录]/seata-cluster/logs/seata2:/root/logs/seata
  seata3:
    image: seataio/seata-server:1.4.2
    hostname: seata3
    container_name: seata3
    ports:
      - "18093:18093"
    networks:
      - nacos-cluster
    environment:
      SEATA_IP: '宿主机IP'
      SEATA_PORT: 18093
      SERVER_NODE: 3
      SEATA_CONFIG_NAME: file:/root/seata-config/registry
      SEATA_ENV: cluster
    volumes:
      - [文件存放目录]/seata-cluster/config:/root/seata-config
      - [文件存放目录]/seata-cluster/logs/seata3:/root/logs/seata
```
* 执行
```shell
docker-compose -f docker-compose.yaml -p seata-cluster up -d
```

### ~~可能遇到的问题(seata-server:1.4.2,since 2021.12.14)~~ 新版本已解决
由于seata在运行时，需要先判断是否运行在容器中，只有运行在容器中时，`SEATA_IP`、`SEATA_PORT`等环境变量才会生效。否则，全部根据Java启动命令来判定。源代码:[io.seata.server.ParameterParser.java](https://github.com/seata/seata/blob/1.4.2/server/src/main/java/io/seata/server/ParameterParser.java)

```java
public class ParameterParser {
    // 省略部分代码
  
  private void init(String[] args) {
    try {
      // 运行在容器中
      if (ContainerHelper.isRunningInContainer()) {
        this.seataEnv = ContainerHelper.getEnv();
        this.host = ContainerHelper.getHost();
        this.port = ContainerHelper.getPort();
        this.serverNode = ContainerHelper.getServerNode();
        this.storeMode = ContainerHelper.getStoreMode();
      } else {
        // Java启动命令
        JCommander jCommander = JCommander.newBuilder().addObject(this).build();
        jCommander.parse(args);
        if (help) {
          jCommander.setProgramName(PROGRAM_NAME);
          jCommander.usage();
          System.exit(0);
        }
      }
      // 省略部分代码
    } catch (ParameterException e) {
      printError(e);
    }
  }
  
  // 省略部分代码
}
```
是否运行在容器是根据`CGROUP`的内容来判断的，源代码:[io.seata.server.ContainerHelper.java](https://github.com/seata/seata/blob/1.4.2/server/src/main/java/io/seata/server/env/ContainerHelper.java)
```java
public class ContainerHelper {

  private static final String C_GROUP_PATH = "/proc/1/cgroup";
  private static final String DOCKER_PATH = "/docker";
  private static final String KUBEPODS_PATH = "/kubepods";

  // 省略部分代码

  public static boolean isRunningInContainer() {
    Path path = Paths.get(C_GROUP_PATH);
    if (Files.exists(path)) {
      try (Stream<String> stream = Files.lines(path)) {
        // 判断路径“/proc/1/cgroup”中是否包含“/docker”字样
        return stream.anyMatch(line -> line.contains(DOCKER_PATH) || line.contains(KUBEPODS_PATH));
      } catch (IOException e) {
        System.err.println("Judge if running in container failed: " + e.getMessage());
        e.printStackTrace();
      }
    }
    return false;
  }

  // 省略部分代码
}
```

而并非所有版本的docker容器，都会在`CGROUP`中定义路径，内容可能为空。由此，引发了一个问题：未定义路径的docker容器，Seata无法成功判断自身运行在容器中，环境变量自然也无法生效。

### 解决方案
* 单机模式  
直接将hostname设置为需要暴露的IP即可注册为对应的IP，只不过端口号依然无法自定义  

* 高可用模式
  以本地开发、测试为例，docker容器都是放在同一台设备上面，所以IP地址都是一样的，只有端口号不一致罢了，这时候就需要解决端口号的问题，纯粹改变hostname已经无法满足。  
  从根本上解决问题的方案之一，就是下载源代码，修改"判断是否运行在容器"的方法，自行编译打包，制作镜像。  
  当前这里提供的就是这个方案(*since 2021.12.14*)：  
  
  1. clone代码到本地，[GitHub地址](https://github.com/seata/seata/tree/1.4.2)
  2. 修改`ContainerHelper#isRunningInContainer`的逻辑为：  
      ```java
      public class ContainerHelper {
      
            private static final String C_GROUP_PATH = "/proc/1/cgroup";
            
            private static final String DOCKER_PATH = "/docker";
            
            private static final String KUBEPODS_PATH = "/kubepods";
            
            // 省略部分代码
            
            private static final String RUN_IN_CONTAINER_KEY = "RUN_IN_CONTAINER";
            
            public static boolean isRunningInContainer() {
              // 新增判断“是否运行在容器中”方法
              if (runInContainer()) {
                return true;
              }
              Path path = Paths.get(C_GROUP_PATH);
              if (Files.exists(path)) {
                try (Stream<String> stream = Files.lines(path)) {
                  return stream.anyMatch(line -> line.contains(DOCKER_PATH) || line.contains(KUBEPODS_PATH));
                } catch (IOException e) {
                  System.err.println("Judge if running in container failed: " + e.getMessage());
                  e.printStackTrace();
                }
              }
              return false;
            }
            
            // 这一段为新增方法，通过读取环境变量“RUN_IN_CONTAINER”来判定是否运行在容器中
            public static boolean runInContainer() {
              return Boolean.TRUE.equals(Boolean.valueOf(System.getenv(RUN_IN_CONTAINER_KEY)));
            }
            
            // 省略部分代码
        }
      ```
  3. 重新编译，获取到[seata-server-1.4.2.jar](./conf/seata/seata-server-1.4.2.jar)  
  4. 从官网下载对应版本的[release包](https://github.com/seata/seata/releases/download/v1.4.2/seata-server-1.4.2.zip) ，并解压获得`seata/seata-server-1.4.2`文件夹  
  5. 使用刚才编译获得的`seata-server-1.4.2.jar`，替换解压文件中的jar包，`seata/seata-server-1.4.2/lib/seata-server-1.4.2.jar`  
  6. 使用[Dockerfile](./conf/seata/Dockerfile)自行构建镜像  
     build镜像命令:`docker build --no-cache -t carzer/seata-server:1.4.2 [dockerfile所在路径]`
  7. 修改原本的docker-compose.yaml文件，增加环境变量`RUN_IN_CONTAINER: true`
       ```yaml
        version: "3.9"
          
        # 直接使用nacos的内部网络
        networks:
          nacos-cluster:
            external: true
            name: nacos-cluster
          
        services:
          seata1:
            image: carzer/seata-server:1.4.2
            hostname: seata1
            container_name: seata1
            ports:
              - "18091:18091"
            networks:
              - nacos-cluster
            environment:
              RUN_IN_CONTAINER: true
              SEATA_IP: '宿主机IP'
              SEATA_PORT: 18091
              SERVER_NODE: 1
              SEATA_CONFIG_NAME: file:/root/seata-config/registry
              SEATA_ENV: cluster
            volumes:
              - [文件存放目录]/seata-cluster/config:/root/seata-config
              - [文件存放目录]/seata-cluster/logs/seata1:/root/logs/seata
          seata2:
            image: carzer/seata-server:1.4.2
            hostname: seata2
            container_name: seata2
            ports:
              - "18092:18092"
            networks:
              - nacos-cluster
            environment:
              RUN_IN_CONTAINER: true
              SEATA_IP: '宿主机IP'
              SEATA_PORT: 18092
              SERVER_NODE: 2
              SEATA_CONFIG_NAME: file:/root/seata-config/registry
              SEATA_ENV: cluster
            volumes:
              - [文件存放目录]/seata-cluster/config:/root/seata-config
              - [文件存放目录]/seata-cluster/logs/seata2:/root/logs/seata
          seata3:
            image: carzer/seata-server:1.4.2
            hostname: seata3
            container_name: seata3
            ports:
              - "18093:18093"
            networks:
              - nacos-cluster
            environment:
              RUN_IN_CONTAINER: true
              SEATA_IP: '宿主机IP'
              SEATA_PORT: 18093
              SERVER_NODE: 3
              SEATA_CONFIG_NAME: file:/root/seata-config/registry
              SEATA_ENV: cluster
            volumes:
              - [文件存放目录]/seata-cluster/config:/root/seata-config
              - [文件存放目录]/seata-cluster/logs/seata3:/root/logs/seata
       ```
  
  8. 执行`docker-compose -f docker-compose.yaml -p seata-cluster up -d`

最后，如果为了能够使用127.0.0.1作为暴露地址(宿主机IP)，需要在修改`ContainerHelper#isRunningInContainer`的同时，额外修改[io.seata.server.Server](https://github.com/seata/seata/blob/1.4.2/server/src/main/java/io/seata/server/Server.java)的内容：  
将代码片段  
```java
public class Server {
  public static void main(String[] args) throws IOException {
      
    // 省略部分代码
    
    // 127.0.0.1 and 0.0.0.0 are not valid here.
    if (NetUtil.isValidIp(parameterParser.getHost(), false)) {
      XID.setIpAddress(parameterParser.getHost());
    } else {
      XID.setIpAddress(NetUtil.getLocalIp());
    }

    // 省略部分代码
  }
}
```
修改为：  
```java
public class Server {
  public static void main(String[] args) throws IOException {
      
    // 省略部分代码
    // 由false变更为true之后，就不会再限制“127.0.0.1”等本机IP作为暴露地址了
    if (NetUtil.isValidIp(parameterParser.getHost(), true)) {
      XID.setIpAddress(parameterParser.getHost());
    } else {
      XID.setIpAddress(NetUtil.getLocalIp());
    }

    // 省略部分代码
  }
}
```


## 15. Nexus

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/nexus/nexus-data
```

* 执行

```shell
docker pull sonatype/nexus3:3.37.0
```

* 创建docker-compose.yaml

```yaml
version: "3.7"
services:
  nexus:
    restart: "no"
    image: sonatype/nexus3:3.37.0
    container_name: nexus
    privileged: true
    ports:
      - "18081:8081"
    volumes:
      - [文件存放目录]/nexus/nexus-data:/nexus-data
```

```shell
docker-compose -f docker-compose.yaml -p nexus up -d
```


## 16. Grafana

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/grafana/data
```

* 执行

```shell
docker pull grafana/grafana:8.3.1
```

```shell
docker run -d --name=grafana -p 3000:3000 \
-v [文件存放目录]/grafana/data:/var/lib/grafana grafana/grafana:8.3.1
```

## 17. Zookeeper Cluster

> 如果没有内部网络 zoo，需要创建一个
> `docker network create zoo --driver bridge`

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/zookeeper/zoo1/conf
mkdir -p [文件存放目录]/zookeeper/zoo1/data
mkdir -p [文件存放目录]/zookeeper/zoo1/datalog

mkdir -p [文件存放目录]/zookeeper/zoo2/conf
mkdir -p [文件存放目录]/zookeeper/zoo2/data
mkdir -p [文件存放目录]/zookeeper/zoo2/datalog

mkdir -p [文件存放目录]/zookeeper/zoo3/conf
mkdir -p [文件存放目录]/zookeeper/zoo3/data
mkdir -p [文件存放目录]/zookeeper/zoo3/datalog
```

* 文件&文件夹准备

[文件存放目录]/zookeeper/zoo1/conf/[zoo.cfg](./conf/zookeeper/zoo.cfg)  
[文件存放目录]/zookeeper/zoo1/conf/[log4j.properties](./conf/zookeeper/log4j.properties)  

[文件存放目录]/zookeeper/zoo2/conf/[zoo.cfg](./conf/zookeeper/zoo.cfg)  
[文件存放目录]/zookeeper/zoo2/conf/[log4j.properties](./conf/zookeeper/log4j.properties)  

[文件存放目录]/zookeeper/zoo3/conf/[zoo.cfg](./conf/zookeeper/zoo.cfg)  
[文件存放目录]/zookeeper/zoo3/conf/[log4j.properties](./conf/zookeeper/log4j.properties)


* 创建docker-compose.yaml

```yaml
version: '3.9'

networks:
  zoo:
    external: true
    name: zoo

services:
  zoo1:
    restart: "no"
    image: zookeeper:3.7.0
    ports:
      - "22181:2181"
    networks:
      - zoo
    container_name: zoo1
    hostname: zoo1
    privileged: true
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - [文件存放目录]/zookeeper/zoo1/conf:/conf
      - [文件存放目录]/zookeeper/zoo1/data:/data
      - [文件存放目录]/zookeeper/zoo1/datalog:/datalog
  zoo2:
    restart: "no"
    image: zookeeper:3.7.0
    ports:
      - "22182:2181"
    networks:
      - zoo
    container_name: zoo2
    hostname: zoo2
    privileged: true
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - [文件存放目录]/zookeeper/zoo2/conf:/conf
      - [文件存放目录]/zookeeper/zoo2/data:/data
      - [文件存放目录]/zookeeper/zoo2/datalog:/datalog
  zoo3:
    restart: "no"
    image: zookeeper:3.7.0
    ports:
      - "22183:2181"
    networks:
      - zoo
    container_name: zoo3
    hostname: zoo3
    privileged: true
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - [文件存放目录]/zookeeper/zoo3/conf:/conf
      - [文件存放目录]/zookeeper/zoo3/data:/data
      - [文件存放目录]/zookeeper/zoo3/datalog:/datalog

```

* 执行：

```shell
docker-compose -f docker-compose.yaml -p zookeeper-cluster  up -d
```

## 18. Kafka Cluster

> 如果没有内部网络 zoo，需要创建一个
> `docker network create zoo --driver bridge`

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/kafka/kafka1/data
mkdir -p [文件存放目录]/kafka/kafka2/data
mkdir -p [文件存放目录]/kafka/kafka3/data

mkdir -p [文件存放目录]/kafka/secrets
```

[文件存放目录]/kafka/secrets/[server_jaas.conf](./conf/kafka/server_jaas.conf)

* 创建docker-compose.yaml

```yaml
version: '3.9'

networks:
  zoo:
    external: true
    name: zoo

services:
  kafka1:
    restart: "no"
    image: bitnami/kafka:3
    container_name: kafka1
    hostname: kafka1
    ports:
      - "29092:29092"
    networks:
      - zoo
    environment:
      KAFKA_BROKER_ID: 1
      ALLOW_PLAINTEXT_LISTENER: yes
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CLIENT:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: CLIENT
      # 同时监听9092与29092，容器网络使用<container>:9092访问kafka，主机网络使用<host>:29092访问kafka
      KAFKA_CFG_LISTENERS: CLIENT://:9092,EXTERNAL://:29092
      KAFKA_CFG_ADVERTISED_LISTENERS: CLIENT://kafka1:9092,EXTERNAL://[外网访问ip]:29092
      KAFKA_CLIENT_USER: admin
      KAFKA_CLIENT_PASSWORD: 'admin_secret'
      # 这里使用的是docker容器内部的端口2181
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    volumes:
      - [文件存放目录]/kafka/kafka1/data:/bitnami/kafka/data
      - [文件存放目录]/kafka/secrets/server_jaas.conf:/opt/bitnami/kafka/config/kafka_jaas.conf:ro
  kafka2:
    restart: "no"
    image: bitnami/kafka:3
    container_name: kafka2
    hostname: kafka2
    ports:
      - "29093:29093"
    networks:
      - zoo
    environment:
      KAFKA_BROKER_ID: 2
      ALLOW_PLAINTEXT_LISTENER: yes
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CLIENT:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: CLIENT
      # 同时监听9092与29092，容器网络使用<container>:9092访问kafka，主机网络使用<host>:29093访问kafka
      KAFKA_CFG_LISTENERS: CLIENT://:9092,EXTERNAL://:29093
      KAFKA_CFG_ADVERTISED_LISTENERS: CLIENT://kafka2:9092,EXTERNAL://[外网访问ip]:29093
      KAFKA_CLIENT_USER: admin
      KAFKA_CLIENT_PASSWORD: 'admin_secret'
      # 这里使用的是docker容器内部的端口2181
      KAFKA_CFG_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    volumes:
      - [文件存放目录]/kafka/kafka2/data:/bitnami/kafka/data
      - [文件存放目录]/kafka/secrets/server_jaas.conf:/opt/bitnami/kafka/config/kafka_jaas.conf:ro
  kafka3:
    restart: "no"
    image: bitnami/kafka:3
    container_name: kafka3
    hostname: kafka3
    ports:
      - "29094:29094"
    networks:
      - zoo
    environment:
      KAFKA_BROKER_ID: 3
      ALLOW_PLAINTEXT_LISTENER: yes
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CLIENT:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: CLIENT
      # 同时监听9092与29092，容器网络使用<container>:9092访问kafka，主机网络使用<host>:29094访问kafka
      KAFKA_CFG_LISTENERS: CLIENT://:9092,EXTERNAL://:29094
      KAFKA_CFG_ADVERTISED_LISTENERS: CLIENT://kafka3:9092,EXTERNAL://[外网访问ip]:29094
      KAFKA_CLIENT_USER: admin
      KAFKA_CLIENT_PASSWORD: 'admin_secret'
      # 这里使用的是docker容器内部的端口2181
      KAFKA_CFG_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    volumes:
      - [文件存放目录]/kafka/kafka3/data:/bitnami/kafka/data
      - [文件存放目录]/kafka/secrets/server_jaas.conf:/opt/bitnami/kafka/config/kafka_jaas.conf:ro
  kafka-manager:
    restart: "no"
    image: sheepkiller/kafka-manager:latest
    container_name: kafka-manager
    hostname: kafka-manager
    ports:
      - "29000:9000"
    links:
      - kafka1
      - kafka2
      - kafka3
    networks:
      - zoo
    environment:
      KM_USERNAME: manager01
      KM_PASSWORD: 'manager01密码'
      ZK_HOSTS: zoo1:2181,zoo2:2181,zoo3:2181
      TZ: CST-8
```

* 执行：

```shell
docker-compose -f docker-compose.yaml -p kafka-cluster  up -d
```

* 可能会用到的命令：

```shell
kafka-console-consumer.sh --bootstrap-server kafka1:9092 --topic [topic_name] --from-beginning
kafka-console-consumer.sh --bootstrap-server kafka1:9092 --topic [topic_name] --property print.timestamp=true --from-beginning |awk -F 'CreateTime:|\t' '$2>= 1637635612260 && $2 <= 1637635613260 {print $0}'
```

## 19. Elasticsearch Cluster

[安装与配置文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/index.html)
> 如果没有内部网络 elastic，需要创建一个
> `docker network create elastic --driver bridge`

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/elasticsearch/es01/data
mkdir -p [文件存放目录]/elasticsearch/es01/plugins
mkdir -p [文件存放目录]/elasticsearch/es01/config
mkdir -p [文件存放目录]/elasticsearch/es01/logs

mkdir -p [文件存放目录]/elasticsearch/es02/data
mkdir -p [文件存放目录]/elasticsearch/es02/plugins
mkdir -p [文件存放目录]/elasticsearch/es02/config
mkdir -p [文件存放目录]/elasticsearch/es02/logs

mkdir -p [文件存放目录]/elasticsearch/es03/data
mkdir -p [文件存放目录]/elasticsearch/es03/plugins
mkdir -p [文件存放目录]/elasticsearch/es03/config
mkdir -p [文件存放目录]/elasticsearch/es03/logs
```

```shell
cd [文件存放目录]/elasticsearch/es0*/config
touch config/elasticsearch.yml
```

elasticsearch.yml内容：

```yaml
cluster.name: "es-docker-cluster"
network.host: 0.0.0.0
```

* 执行

```shell
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.16.2
```

* 创建docker-compose.yaml

```yaml
version: '3.9'

networks:
  elastic:
    external: true
    name: elastic

services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.2
    container_name: es01
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elastic
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - [文件存放目录]/elasticsearch/es01/data:/usr/share/elasticsearch/data
      - [文件存放目录]/elasticsearch/es01/logs:/usr/share/elasticsearch/logs
      - [文件存放目录]/elasticsearch/es01/plugins:/usr/share/elasticsearch/plugins
      - [文件存放目录]/elasticsearch/es01/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.2
    container_name: es02
    ports:
      - "9210:9200"
      - "9310:9300"
    networks:
      - elastic
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - [文件存放目录]/elasticsearch/es02/data:/usr/share/elasticsearch/data
      - [文件存放目录]/elasticsearch/es02/logs:/usr/share/elasticsearch/logs
      - [文件存放目录]/elasticsearch/es02/plugins:/usr/share/elasticsearch/plugins
      - [文件存放目录]/elasticsearch/es02/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.2
    container_name: es03
    ports:
      - "9220:9200"
      - "9320:9300"
    networks:
      - elastic
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - [文件存放目录]/elasticsearch/es03/data:/usr/share/elasticsearch/data
      - [文件存放目录]/elasticsearch/es03/logs:/usr/share/elasticsearch/logs
      - [文件存放目录]/elasticsearch/es03/plugins:/usr/share/elasticsearch/plugins
      - [文件存放目录]/elasticsearch/es03/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
```

* 执行：

```shell
docker-compose -f docker-compose.yaml -p elasticsearch-cluster up -d
```

### 开启权限认证

1. 在宿主机执行  
```shell
# 创建CA(certificate authority)
docker exec -it es01 bin/elasticsearch-certutil ca
# 创建证书
# 需要output file时，输入：/usr/share/elasticsearch/config/elastic-certificates.p12
docker exec -it es01 bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
# 将证书复制到宿主机，docker目前不支持容器间互拷
docker cp es01:/usr/share/elasticsearch/config/elastic-certificates.p12 [文件存放目录]/elasticsearch
# 复制证书到其他节点
docker cp [文件存放目录]/elasticsearch es02:/usr/share/elasticsearch/config/elastic-certificates.p12
docker cp [文件存放目录]/elasticsearch es03:/usr/share/elasticsearch/config/elastic-certificates.p12
```
另外，这里提供已经创建好的[证书文件](./conf/elasticsearch/elastic-certificates.p12)，可直接映射到对应的容器中  
在非指定文件的情况下，需要变更一下证书的权限`chmod 644 elastic-certificates.p12`

2. 修改elasticsearch.yml内容：  
```yaml
cluster.name: "es-docker-cluster"
network.host: 0.0.0.0
# 开启权限认证
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
# 如果elastic-certificates.p12文件和当前的配置文件不在同一目录（也就是/usr/share/elasticsearch/config），则需要写绝对路径
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```
3. 重新启动集群
4. 设置密码
```shell
docker exec -it es01 bin/elasticsearch-setup-passwords auto
```
> 这个命令会自动生成几个默认用户和随机密码。  
如果想手动生成密码，则使用 `docker exec -it es01 bin/elasticsearch-setup-passwords interactive` 命令。  
一般默认会生成几个管理员账户，其中一个叫elastic的用户是超级管理员。  

密码如下：
```
Changed password for user apm_system
PASSWORD apm_system = 

Changed password for user kibana_system
PASSWORD kibana_system = 

Changed password for user kibana
PASSWORD kibana = 

Changed password for user logstash_system
PASSWORD logstash_system = 

Changed password for user beats_system
PASSWORD beats_system = 

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = 

Changed password for user elastic
PASSWORD elastic = 
```
保存密码备用，如果同时使用kibana，建议使用**kibana_system**用户进行连接

### 万一忘记密码

1. 创建临时超级用户
```shell
docker exec -it es01 /bin/elasticsearch-users useradd s_elastic -r superuser
# 输入密码
s_elastic_pass
```
2. 调用API修改elastic密码
```shell
curl -XPUT -u s_elastic:s_elastic_pass http://localhost:9200/_xpack/security/user/elastic/_password -H "Content-Type: application/json" -d '
{
  "password": "passw0rd"
}'
```

## 20. Kibana

[安装与配置文档](https://www.elastic.co/guide/en/kibana/7.16/index.html)
> 如果没有内部网络 elastic，需要创建一个
> `docker network create elastic --driver bridge`

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/kibana/config
mkdir -p [文件存放目录]/kibana/data
```

```shell
cd [文件存放目录]/kibana/config
touch config/kibana.yml
```

kibana.yml内容：

```yaml
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
monitoring.ui.container.elasticsearch.enabled: true
i18n.locale: "zh-CN"
```

* 执行

```shell
docker pull docker.elastic.co/kibana/kibana:7.16.2
```

* 创建docker-compose.yaml

```yaml
version: "3.9"

networks:
  elastic:
    external: true
    name: elastic

services:
  kibana:
    image: docker.elastic.co/kibana/kibana:7.16.2
    container_name: kibana
    ports:
      - "5601:5601"
    networks:
      - elastic
    environment:
      ELASTICSEARCH_HOSTS: '["http://es01:9200","http://es02:9200","http://es03:9200"]'
    volumes:
      - [文件存放目录]/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml
      - [文件存放目录]/kibana/data:/usr/share/kibana/data
```

* 执行：

```shell
docker-compose -f docker-compose.yaml -p kibana up -d
```

### 开启权限认证

如果es开启了权限认证，kibana需要同步进行调整
1. 修改kibana.yml内容  
```yaml
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
monitoring.ui.container.elasticsearch.enabled: true
i18n.locale: "zh-CN"
# 连接es所使用的用户名
elasticsearch.username: "kibana_system"
```
2. 启动kibana
3. 创建keystore
```shell
docker exec -it kibana bin/kibana-keystore create
```
4. 添加刚才设置的kibana_system用户的密码
```shell
docker exec -it kibana bin/kibana-keystore add elasticsearch.password
passw0rd
```

## 21. HBase

### 单机、丐版搭建说明  

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/hbase/data
```

* 执行
```shell
docker pull harisekhon/hbase:2.1
```

```shell
docker run --name hbase -h hbase-server -p 16000:16000 \
-p 16010:16010 \
-p 16020:16020 \
-p 16030:16030 \
-p 16201:16201 \
-p 16301:16301 \
-p 32181:2181 \
-p 38080:8080 \
-p 38085:8085 \
-p 39090:9090 \
-p 39095:9095 \
-v [文件存放目录]/hbase/data:/hbase-data \
--restart=always -d harisekhon/hbase:2.1
```

> 由于HBase自身的种种原因，需要修改访问者的host文件`/etc/hosts`，将`hbase-server`配置为HBase容器宿主机的IP  
> 例如[大数据服务](./carzer-bigdata)build的镜像，在运行`docker run`命令时，需要增加`--add-host='hbase-server:[HBase宿主机ip]'`

### 集群模式

集群模式因为没有特别合适的镜像，所以采用自行搭建的方案，所有涉及到zookeeper的地方，均使用了外部zookeeper  

* 文件&文件夹准备  

  下载
  - [hadoop-3.3.1](https://downloads.apache.org/hadoop/common/hadoop-3.3.1/hadoop-3.3.1.tar.gz)  
  - [hbase-2.4.9](https://dlcdn.apache.org/hbase/2.4.9/hbase-2.4.9-bin.tar.gz )

  创建文件夹
```shell
mkdir -p [文件存放目录]/hbase-cluster/master
mkdir -p [文件存放目录]/hbase-cluster/region01
mkdir -p [文件存放目录]/hbase-cluster/region02
mkdir -p [文件存放目录]/hbase-cluster/backup
mkdir -p [文件存放目录]/hbase-cluster/tmp
```

将工程中的 [配置文件](./conf/hbase) 复制到对应的文件夹中  

使用openjdk8的镜像作为基础镜像  
```shell
docker pull openjdk:8-jdk-bullseye
```

使用[Dockerfile](./conf/hbase/Dockerfile)构建镜像:`docker build --no-cache -t carzer/hbase:1  [dockerfile所在路径]`，镜像中搭建了hdfs与hbase的集群  

* 创建docker-compose.yaml

```yaml
version: "3.9"

networks:
  zoo:
    external: true
    name: zoo

services:
  hbase-master:
    image: carzer/hbase:1
    hostname: hbase-master
    container_name: hbase-master
    privileged: true
    networks:
      - zoo
    ports:
      - '50070:50070'
      - '8088:8088'
      - '16000:16000'
      - '16010:16010'
    volumes:
      - [文件存放目录]/hbase-cluster/master/start.sh:/root/start.sh
      - [文件存放目录]/hbase-cluster/master/logs/hbase:/usr/local/hbase-2.4.9/logs
      - [文件存放目录]/hbase-cluster/master/logs/hadoop:/usr/local/hadoop-3.3.1/logs
      - [文件存放目录]/hbase-cluster/master/conf/hbase-site.xml:/usr/local/hbase/conf/hbase-site.xml
      - [文件存放目录]/hbase-cluster/master/conf/yarn-site.xml:/usr/local/hadoop/etc/hadoop/yarn-site.xml
      # mac环境下，因为文件系统不支持，所以需要创建卷进行挂载：docker volume create hdfs-master
      - hdfs-master:/root/hdfs
      # 其他环境可直接外挂数据
      - [文件存放目录]/master/hdfs:/root/hdfs
    environment:
      name: value
    deploy:
      restart_policy:
        condition: on-failure
    links:
      - hbase-region01
      - hbase-region02
      - hbase-backup
  hbase-region01:
    image: carzer/hbase:1
    hostname: hbase-region01
    container_name: hbase-region01
    privileged: true
    networks:
      - zoo
    ports:
      - '26020:26020'
      - '26030:26030'
    volumes:
      - [文件存放目录]/hbase-cluster/region01/start.sh:/root/start.sh
      - [文件存放目录]/hbase-cluster/region01/logs/hbase:/usr/local/hbase/logs
      - [文件存放目录]/hbase-cluster/region01/conf/hbase-site.xml:/usr/local/hbase/conf/hbase-site.xml
      - [文件存放目录]/hbase-cluster/region01/conf/yarn-site.xml:/usr/local/hadoop/etc/hadoop/yarn-site.xml
    deploy:
      restart_policy:
        condition: on-failure
  hbase-region02:
    image: carzer/hbase:1
    hostname: hbase-region02
    container_name: hbase-region02
    privileged: true
    networks:
      - zoo
    ports:
      - '36020:36020'
      - '36030:36030'
    volumes:
      - [文件存放目录]/hbase-cluster/region02/start.sh:/root/start.sh
      - [文件存放目录]/hbase-cluster/region02/logs/hbase:/usr/local/hbase/logs
      - [文件存放目录]/hbase-cluster/region02/conf/hbase-site.xml:/usr/local/hbase/conf/hbase-site.xml
      - [文件存放目录]/hbase-cluster/region02/conf/yarn-site.xml:/usr/local/hadoop/etc/hadoop/yarn-site.xml
    deploy:
      restart_policy:
        condition: on-failure
    links:
      - hbase-region01
  hbase-backup:
    image: carzer/hbase:1
    hostname: hbase-backup
    container_name: hbase-backup
    privileged: true
    networks:
      - zoo
    ports:
      - '16001:16001'  
      - '16011:16011'
    volumes:
      - [文件存放目录]/hbase-cluster/backup/start.sh:/root/start.sh
      - [文件存放目录]/hbase-cluster/backup/logs/hbase:/usr/local/hbase-2.4.9/logs
      - [文件存放目录]/hbase-cluster/backup/logs/hadoop:/usr/local/hadoop-3.3.1/logs
      - [文件存放目录]/hbase-cluster/backup/conf/hbase-site.xml:/usr/local/hbase/conf/hbase-site.xml
      - [文件存放目录]/hbase-cluster/backup/conf/yarn-site.xml:/usr/local/hadoop/etc/hadoop/yarn-site.xml
      # mac环境下，因为文件系统不支持，所以需要创建卷进行挂载：docker volume create hdfs-backup
      # - hdfs-backup:/root/hdfs
      # 其他环境可直接外挂数据
      - [文件存放目录]/backup/hdfs:/root/hdfs
    deploy:
      restart_policy:
        condition: on-failure
    links:
      - hbase-region01
      - hbase-region02
volumes:
  hdfs-master:
    external: true
  hdfs-backup:
    external: true
```

* 执行

```shell
docker-compose -f docker-compose.yaml -p hbase-cluster up -d
```

* 可能用到的信息  

hbase集群状态查看地址：http://localhost:16010
hdfs集群状态查看地址：http://localhost:50070/

* 需要注意的地方  

master和backup在初次启动时，有格式化的操作，初次启动结束后，需要根据启动脚本中的注释，对相应的命令注释或打开注释。  
同样的，需要修改访问者的host文件`/etc/hosts`，将yaml文件中的hostname配置为HBase容器宿主机的IP，例：  
```shell
127.0.0.1		hbase-master
127.0.0.1		hbase-backup
127.0.0.1		hbase-region01
127.0.0.1		hbase-region02
```

> 参考文档：https://www.cnblogs.com/latiaotaba/p/10180099.html，https://blog.csdn.net/weixin_44966780/article/details/121769739

## 22. GitLab

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/gitlab/etc
mkdir -p [文件存放目录]/gitlab/log
mkdir -p [文件存放目录]/gitlab/opt
```

* 执行

```shell
docker pull gitlab/gitlab-ce:14.5.0-ce.0
```

```shell
docker run \
 -itd  \
 -p 48080:80 \
 -p 48022:22 \
 -e GITLAB_OMNIBUS_CONFIG="external_url 'http://localhost:48080'; gitlab_rails['gitlab_shell_ssh_port'] = 48022; nginx['listen_addresses'] = ['*']; nginx['listen_port'] = 80;" \
 -v [文件存放目录]/gitlab/etc:/etc/gitlab  \
 -v [文件存放目录]/gitlab/log:/var/log/gitlab \
 -v [文件存放目录]/gitlab/opt:/var/opt/gitlab \
 --restart always \
 --privileged=true \
 --name gitlab \
 --shm-size 256m \
 gitlab/gitlab-ce:15.2.2-ce.0
```
> 关键参数说明  
> external_url 'http://localhost:48080'  *# GitLab对外开放的工程地址前缀*  
> nginx['listen_port'] = 80  *# 内嵌的nginx需要监听的端口是(-p 48080:80)中的80，如果不修改，内嵌nginx的监听端口会是48080，导致无法正常访问*


查看root用户的默认密码
```shell
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

## 23. SVN

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/svn
```

* 执行

```shell
docker pull garethflowers/svn-server:1.6.0
```

```shell
docker run  \
  -v [文件存放目录]/svn:/var/opt/svn \
  --name svn-server \
  -p 13690:3690 \
  --privileged=true \
  --restart=always \
  -e SVN_REPONAME=repository \
  -d docker.io/garethflowers/svn-server:1.6.0
```
用户-密码管理文件  
`[文件存放目录]/svn/repository/conf/passwd`  
权限管理文件  
`[文件存放目录]/svn/repository/conf/authz`  
[参考](./conf/svn)

## 24. ZenTao

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/zentao
```

* 执行

```shell
docker pull easysoft/zentao:15.0.3
```

```shell
docker run \
--name=zentao \
--env='MYSQL_ROOT_PASSWORD=root密码' \
--volume=[文件存放目录]/zentao/zentaopms:/www/zentaopms \
--volume=[文件存放目录]/zentao/mysql:/var/lib/mysql \
-p 38080:80 \
-d easysoft/zentao:15.0.3
```
如果需要使用外部mysql，可以编辑`my.php`文件，修改具体的数据库连接信息,文件位置`[文件存放目录]/zentao/zentaopms/config/my.php`


## 25. ZipKin
(施工中...)  

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/zipkin
```

* 执行

```shell
docker pull openzipkin/zipkin:2.23.9
```
简单部署
```shell
docker run -d \
--restart always \
-v /etc/localtime:/etc/localtime:ro \
-p 9411:9411 \
--name zipkin \
openzipkin/zipkin:2.23.9
```
docker-compose部署，存储到ES
```yaml
version: '3.9'

networks:
  elastic:
    external: true
    name: elastic

services:
  zipkin:
    image: openzipkin/zipkin:2.23.9
    container_name: zipkin
    networks:
      - elastic
    environment:
      STORAGE_TYPE: elasticsearch
      # Point the zipkin at the storage backend
      ES_HOSTS: es01:9200,es02:9200,es03:9200
      # Uncomment to enable scribe
      # - SCRIBE_ENABLED=true
      # Uncomment to enable self-tracing
      # - SELF_TRACING_ENABLED=true
      # Uncomment to enable debug logging
      JAVA_OPTS: -Dlogging.level.zipkin=DEBUG -Dlogging.level.zipkin2=DEBUG
    ports:
      # Port used for the Zipkin UI and HTTP Api
      - '19411:9411'
      # Uncomment if you set SCRIBE_ENABLED=true
      # - 9410:9410
```
* 执行：

```shell
docker-compose -f docker-compose.yaml -p zipkin up -d
```
[yaml配置参考](https://github.com/openzipkin-attic/docker-zipkin)

## 26. logstash

施工中...

```shell
docker pull logstash:7.14.2
```

## 27. rancher

* 文件&文件夹准备

```shell
mkdir -p [文件存放目录]/rancher2/rancher
mkdir -p [文件存放目录]/rancher2/auditlog
```

* 执行
```shell
docker pull rancher/rancher:v2.6-head
```

```shell
docker run -d \
--restart always \
-v /etc/localtime:/etc/localtime:ro \
-p 18080:80 -p 18443:443 \
-v [文件存放目录]/rancher/rancher:/var/lib/rancher \
-v [文件存放目录]/rancher/auditlog:/var/log/auditlog \
--name rancher \
--privileged \
rancher/rancher:v2.6-head
```



# 部分命令解释

以gitlab为例的命令解释：
>-i  以交互模式运行容器，通常与 -t 同时使用命令解释：  
-t  为容器重新分配一个伪输入终端，通常与 -i 同时使用  
-d  后台运行容器，并返回容器ID  
-p 48080:80  将容器内80端口映射至宿主机48080端口，web端口  
-p 48022:22  将容器内22端口映射至宿主机48022端口，ssh端口  
-v /usr/local/gitlab/etc:/etc/gitlab  将容器/etc/gitlab目录挂载到宿主机[文件存放目录]/gitlab/etc下，若宿主机内此目录不存在将会自动创建  
--restart always  容器自启动  
--privileged=true  让容器获取宿主机root权限  
--name gitlab  设置容器名称为gitlab  
gitlab/gitlab-ce:14.5.0-ce.0  镜像的名称，这里也可以写镜像ID  

查询容器IP：
> docker inspect gitlab | grep IPAddress  
