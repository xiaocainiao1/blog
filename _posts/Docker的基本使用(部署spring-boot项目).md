# Docker的基本使用(部署spring-boot项目)

今天开始利用docker来部署项目，当然，首先，需要安装好Docker，这个在之前的PPT讲过。

## 1.将springboot项目打成jar包

![springboot项目打成的jar包打成镜像放到docker中](https://images1.tqwba.com/20200913/yvpiz0k01sa.png)

## 2.创建一个Dockerfile文件

```dockerfile
FROM java:8   --基础镜像,根据业务的实际情况进行选择
ADD *.jar /app.jar --把当前包复制到镜像中
CMD ["--server.port=8080"] --指定启动时服务器绑定端口
EXPOSE 8080 --声明端口 
ENTRYPOINT ["java","-jar","/app.jar"]
```

## 3.将jar包和Dockerfile文件放到同一目录中

![springboot项目打成的jar包打成镜像放到docker中](https://images1.tqwba.com/20200913/zu5o2afmyed.png)

## 4.打成镜像：

```
docker build -t blog .
```

## 5.运行镜像

```
docekr run -d -p 8080:8080 blog
```



## 六、docker的导入和导出

```
docker 镜像导入导出有两种方法：

一种是使用 save 和 load 命令

使用例子如下：

docker save blog:latest>/root/blog.tar
docker load<blog.tar

一种是使用 export 和 import 命令

使用例子如下：

docker export blog:latest> blog.tar
cat blog.tar |  docker import - blog:latest

export 和 import 导出的是一个容器的快照, 不是镜像本身, 也就是说没有 layer。

你的 dockerfile 里的 workdir, entrypoint 之类的所有东西都会丢失，commit 过的话也会丢失。

快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也更大。

 docker save 保存的是镜像（image），docker export 保存的是容器（container）；
 docker load 用来载入镜像包，docker import 用来载入容器包，但两者都会恢复为镜像；
 docker load 不能对载入的镜像重命名，而 docker import 可以为镜像指定新名称。
```

