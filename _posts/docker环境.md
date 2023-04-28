#                                                                            docker环境和命令

## 1. docker 远程api 配置

本次实战中，IDEA作为开发电脑，要远程连接到另一台Linux电脑上部署的Docker服务192.168.2.95，我们登陆到机器上，进行docker远程访问配置：

```
# vi /usr/lib/systemd/system/docker.service 
修改：
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock --containerd=/run/containerd/containerd.sock
```

重启docker 服务：

```
[root@dev_vonedao_95 ~]# systemctl daemon-reload
[root@dev_vonedao_95 ~]# systemctl restart docker
[root@dev_vonedao_95 ~]# ps -ef |grep docker
root     12319     1  1 18:27 ?        00:00:00 /usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock --containerd=/run/containerd/containerd.sock
root     12461 12217  0 18:27 pts/0    00:00:00 grep --color=auto docker
```

## 2.docker私服务构建需要注意的修改

修改默认的镜像地址为阿里云 ，daemon.json没有就新建

```
# vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://gp4b3g57.mirror.aliyuncs.com"]
}


```

dokcer 默认不允许http请求的，docker私服的地址需要配置

```
# vim /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://gp4b3g57.mirror.aliyuncs.com"
  ],
  "insecure-registries": [
    "http://39.100.127.235:8009"
  ]
}
```

推送到私服和从私服务器拉取

```
# 登陆远程仓库
docker login   39.100.127.235:8009/docker login -u grid -p grid@123 172.17.66.165:8009  #注意这里的端口是配置仓库时选择的端口号
# 拉取镜像
docker pull openjdk:8-jdk-alpine
# 私服标签
docker tag openjdk:8-jdk-alpine  39.100.127.235:8009/openjdk:8-jdk-alpine
# 推送至私服
docker push 39.100.127.235:8009/openjdk:8-jdk-alpine
# 搜索镜像
docker search 39.100.127.235:8009/openjdk:8-jdk-alpine
# 退出
docker logout
```

## 3.docker导出和导入

```
docker 镜像导入导出有两种方法：

一种是使用 save 和 load 命令

使用例子如下：

docker save ubuntu:load>/root/ubuntu.tar
docker load<ubuntu.tar

一种是使用 export 和 import 命令

使用例子如下：

docker export 98ca36> ubuntu.tar
cat ubuntu.tar | sudo docker import - ubuntu:import

export 和 import 导出的是一个容器的快照, 不是镜像本身, 也就是说没有 layer。

你的 dockerfile 里的 workdir, entrypoint 之类的所有东西都会丢失，commit 过的话也会丢失。

快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也更大。

 docker save 保存的是镜像（image），docker export 保存的是容器（container）；
 docker load 用来载入镜像包，docker import 用来载入容器包，但两者都会恢复为镜像；
 docker load 不能对载入的镜像重命名，而 docker import 可以为镜像指定新名称。
```

## 4.docker 重命令

```
docker tag IMAGEID(镜像id) REPOSITORY:TAG（仓库：标签）

#例子
docker tag ca1b6b825289 registry.cn-hangzhou.aliyuncs.com/xxxxxxx:v1.0 
```

如果名字，版本已存在，重命名，会生成新的，不过会引用老的

## 5.docker 容器已启动的情况修改随着docker 自动启动或者暴露端口

容器自启动分为2种情况：

1. 新建容器时配置自启动参数

   ```
   docker run --restart=always 镜像id或者镜像名称
   ```

2. 为存在的容器配置自启动

   ```
   docker update --restart=no 容器id或者容器名称
   docker update --restart=always ${docker ps -aq}
   ```

3. 修改配置文件

   ```
   /var/lib/docker/containers/[hash_of_the_container]/hostconfig.json  
   /var/lib/docker/containers/[hash_of_the_container]/config.v2.json
   cd /var/lib/docker/containers/81ac47………….272e   --对应的容器ID
   vim hostconfig.json
   ```

Docker添加和修改运行中的容器的端口的方法

- 背景
  有时候我们创建好了一个容器，在容器内部放置了一些服务，在某天新增一个服务的时候，发现之前并没有创建该端口的映射，导致在容器之外的服务访问不可达。但是docker 仅在run的时候提供了-p参数来增加端口映射，也没有start的相关修改端口映射的方式，那么要暴露服务大概可以采取以下方式。

​         1.重新创建一个容器，增加暴露端口，重新部署原来的服务
​         2.将容器重新commit为镜像，再次运行成一个新容器，然后指定暴露端口
​         3.修改配置文件的方式
​         4.由于第一种和第二种操作比较繁琐，本文针对第三种方式做一个简要介绍。

- 修改配置文件的步骤
  1.停止容器（特别重要）
  在执行以下步骤之前 一定要先停止容器

  ```
  docker stop 容器id/容器名称
  ```

  2.在宿主机修改容器配置文件配置文件的位置如下，一共要修改两个配置文件

  ```
  /var/lib/docker/containers/[hash_of_the_container]/hostconfig.json  
  /var/lib/docker/containers/[hash_of_the_container]/config.v2.json 
  ```

  路径里面的[hash_of_the_container]通过下面命令查看

  ```
  docker inspect 容器ID/容器名称
  ```

  首先修改hostconfig.json如我的地址

  ```
  vi /var/lib/docker/containers/8b8e2f696a9c89e8c4d4df061b10e69b94975c73db36b63514df19967ecf113b/hostconfig.json
  ```

  看到的效果如下

  ![image-20210415104727544](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210415104727544.png)我们找到PortBindings关键字（如果没有就增加），里面有我们之前添加的端口映射形式，在该节点里面仿照形式再添加一组端口

  我添加的

   

  ```
  "9200/tcp":[{"HostIp":"","HostPort":"19200"}],"9300/tcp":[{"HostIp":"","HostPort":"19300"}]
  ```

  插入后的效果

  ![image-20210415104840897](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210415104840897.png)

  再修改配置文件config.v2.json暴露端口

  ```
  vi /var/lib/docker/containers/8b8e2f696a9c89e8c4d4df061b10e69b94975c73db36b63514df19967ecf113b/config.v2.json
  ```

  在config节点里面增加一个 （如果没有ExposedPorts的话），如果存在了，直接在ExposedPorts里面仿照已有格式暴露需要暴露的端口

  ```
  ###我的文件已经存在下列端口 
  
  ExposedPorts:{"22/tcp":{},"9002/tcp":{},"9003/tcp":{}}
  
  ##增加需要额外暴露的9200 和93000端口 
  
  ExposedPorts:{"22/tcp":{},"9002/tcp":{},"9003/tcp":{},"9200/tcp":{},"9300/tcp":{}}
  ```

  增加后的效果

  ![image-20210415105016862](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210415105016862.png)

3.重启docker服务（在这之前不要启动容器）

如果改动文件后启动了容器，那么你的配置又会被刷新为之前的配置了（你只好从第一步再来一遍）

```
systemctl docker restart
```

启动容器
功成身退 

注意：此时虽然在**docker ps -a**看不到新增的暴露端口，但是通过配置文件和实际访问均可验证配置成功了

## 6.docker 提交容器

```
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
docker commit -a "runoob.com" -m "my apache" a404c6c174a2  mymysql:v1
```

## 7.docker 删除无用的容器或者镜像

```
docker rm `docker ps -a | grep Exited | awk '{print $1}'`   删除异常停止的docker容器

docker rmi -f  `docker images | grep '<none>' | awk '{print $3}'`  删除名称或标签为none的镜像
```

## 8.docker 容器信息完整展示

```
docker ps --no-trunc   -- 展示完整内容
```

## 9.centos7 docker配置TLS认证的远程端口的证书生成教程（shell脚本一键生成）

1. 创建运行脚本 

   ```
   #创建 Docker TLS 证书
   #!/bin/bash
   
   #相关配置信息
   SERVER="192.168.33.76"
   PASSWORD="pass123456"
   COUNTRY="CN"
   STATE="广州省"
   CITY="广州市"
   ORGANIZATION="公司名称"
   ORGANIZATIONAL_UNIT="Dev"
   EMAIL="492376344@qq.com"
   
   ###开始生成文件###
   echo "开始生成文件"
   
   #切换到生产密钥的目录
   cd /etc/docker   
   #生成ca私钥(使用aes256加密)
   openssl genrsa -aes256 -passout pass:$PASSWORD  -out ca-key.pem 2048
   #生成ca证书，填写配置信息
   openssl req -uf8 -new -x509 -passin "pass:$PASSWORD" -days 3650 -key ca-key.pem -sha256 -out ca.pem -subj "/C=$COUNTRY/ST=$STATE/L=$CITY/O=$ORGANIZATION/OU=$ORGANIZATIONAL_UNIT/CN=$SERVER/emailAddress=$EMAIL"
   
   #生成server证书私钥文件
   openssl genrsa -out server-key.pem 2048
   #生成server证书请求文件
   openssl req -subj "/CN=$SERVER" -new -key server-key.pem -out server.csr
   #使用CA证书及CA密钥以及上面的server证书请求文件进行签发，生成server自签证书
   openssl x509 -req -days 3650 -in server.csr -CA ca.pem -CAkey ca-key.pem -passin "pass:$PASSWORD" -CAcreateserial  -out server-cert.pem
   
   #生成client证书RSA私钥文件
   openssl genrsa -out key.pem 2048
   #生成client证书请求文件
   openssl req -subj '/CN=client' -new -key key.pem -out client.csr
   
   sh -c 'echo "extendedKeyUsage=clientAuth" > extfile.cnf'
   #生成client自签证书（根据上面的client私钥文件、client证书请求文件生成）
   openssl x509 -req -days 3650 -in client.csr -CA ca.pem -CAkey ca-key.pem  -passin "pass:$PASSWORD" -CAcreateserial -out cert.pem  -extfile extfile.cnf
   
   #更改密钥权限
   chmod 0400 ca-key.pem key.pem server-key.pem
   #更改密钥权限
   chmod 0444 ca.pem server-cert.pem cert.pem
   #删除无用文件
   rm client.csr server.csr
   
   echo "生成文件完成"
   ###生成结束###
   
   ```

   

2. 执行结束我们可以在/etc/docker 目录下看到相应的证书文件

   ![image-20210414185703882](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210414185703882.png)

3.配置证书

```
方式一:配置daemon.json
/etc/docker/daemon.json 
{
  "registry-mirrors": ["http://hub-mirror.c.163.com"],
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"],
  "tlsverify": true,
  "tlscacert": "/root/.docker/ca.pem",
  "tlscert": "/root/.docker/server.pem",
  "tlskey": "/root/.docker/server-key.pem"
}
方式二:配置docker.service
vi /usr/lib/systemd/system/docker.service
在ExecStart属性后面追加（我们刚刚自己生成的证书路径）
(配置 docker 使用 TLS 认证并监听 tcp 端口)
--tlsverify \
--tlscacert=/etc/docker/ca.pem \
--tlscert=/etc/docker/server-cert.pem \
--tlskey=/etc/docker/server-key.pem \
-H tcp://0.0.0.0:2376 \
-H unix:///var/run/docker.sock 


```

4.重启docker 让配置生效

```
[root@dev_vonedao_95 ~]# systemctl daemon-reload
[root@dev_vonedao_95 ~]# systemctl restart docker
```

5.验证

方式一：

```
curl -k https://127.0.0.1:2376/images/json --cert /etc/docker/cert.pem --key /etc/docker/key.pem --cacert /etc/docker/ca.pem
或者
curl -k https://127.0.0.1:2376/images/json \
      --cert /etc/docker/cert.pem \
      --key /etc/docker/key.pem \
      --cacert /etc/docker/ca.pem

```

方式二：

```
docker --tls -H 192.168.100.13 ps
```

