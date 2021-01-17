                                         **docker环境**

###### 1. docker 远程api 配置

本次实战中，IDEA作为开发电脑，要远程连接到另一台Linux电脑上部署的Docker服务192.168.2.95，我们登陆到机器上，进行docker远程访问配置：

```
# vi usr/lib/systemd/system/docker.service 
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

###### 2.docker私服务构建需要注意的修改

- 修改默认的镜像地址为阿里云 ，daemon.json没有就新建

  ```
  # vim /etc/docker/daemon.json
  {
    "registry-mirrors": ["https://gp4b3g57.mirror.aliyuncs.com"]
  }
  
  
  ```

- dokcer 默认不允许http请求的，docker私服的地址需要配置

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

-  推送到私服

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
  ```
  
  