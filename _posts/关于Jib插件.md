#                      **Jib使用小结(Maven插件版)**

近期在用Jib插件将Java工程构建成Docker镜像，使用时遇到过一些小问题，今天对这些问题做个小结；

### 关于Jib插件

Jib是用于构建Docker镜像的Maven插件，其基本用法请参考[《Docker与Jib(maven插件版)实战》](https://blog.csdn.net/boling_cavalry/article/details/94355659)一文；

### 全文概览

本文由以下几部分组成：

1. 环境信息
2. 源码下载
3. 小结一：三种构建参数
4. 小结二：镜像的时间问题
5. 小结三：多次构建后，积累的无用镜像问题
6. 小结四：提升构建速度
7. 小结五：将jib与mvn构建的生命周期绑定
8. 小结六：父子结构的maven工程如何构建

### 环境信息

1. 操作系统：CentOS Linux release 7.6.1810
2. docker：1.13.1
3. jdk：1.8.0_191
4. maven：3.6.0
5. jib插件：1.3.0

### 源码下载

本次实战用到的源码是个简单的maven工程，可以从GitHub上下载本次实战的源码，地址和链接信息如下表所示：

| 名称               | 链接                                     | 备注                            |
| ------------------ | ---------------------------------------- | ------------------------------- |
| 项目主页           | https://github.com/zq2599/blog_demos     | 该项目在GitHub上的主页          |
| git仓库地址(https) | https://github.com/zq2599/blog_demos.git | 该项目源码的仓库地址，https协议 |
| git仓库地址(ssh)   | git@github.com:zq2599/blog_demos.git     | 该项目源码的仓库地址，ssh协议   |

这个git项目中有多个文件夹，本章的源码在hellojib文件夹下，如下图红框所示：
![image-20210105104246306](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210105104246306.png)

### 小结一：三种构建参数

对于一个已在pom.xml中配置了jib插件的java工程来说，下面是个标准的构建命令：

```shell
mvn compile jib:dockerBuild
```

注意上面的dockerBuild参数，该参数的意思是将镜像存入当前的镜像仓库，这样的参数一共有三种，列表说明：

| 参数名      | 作用                                                         |
| ----------- | ------------------------------------------------------------ |
| dockerBuild | 将镜像存入当前镜像仓库，该仓库是当前docker客户端可以连接的docker daemon，一般是指本地镜像仓库 |
| build       | 将镜像推送到远程仓库，仓库位置与镜像名字的前缀有关，一般是hub.docker.com，使用该参数时需要提前登录成功 |
| buildTar    | 将镜像生成tar文件，保存在项目的target目录下，在任何docker环境执行 docker load --input xxx.tar即可导入到本地镜像仓库 |

### 小结二：镜像的时间问题

在使用命令mvn compile jib:dockerBuild构建本地镜像时，会遇到创建时间不准的问题：
如下所示，bolingcavalry/hellojib是刚刚使用jib插件构建的镜像，其生成时间(CREATED字段)显示的是49 years ago：

```shell
[root@maven hellojib]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
bolingcavalry/hellojib   0.0.1-SNAPSHOT      ef96fdd4473a        49 years ago        505 MB
123
```

上面显示的镜像生成时间显然是不对的，改正此问题的方法是修改pom.xml，在jib插件的container节点内增加useCurrentTimestamp节点，内容是true，如下所示：

```xml
<plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>1.3.0</version>
                <configuration>
                    <!--from节点用来设置镜像的基础镜像，相当于Docerkfile中的FROM关键字-->
                    <from>
                        <!--使用openjdk官方镜像，tag是8-jdk-stretch，表示镜像的操作系统是debian9,装好了jdk8-->
                        <image>openjdk:8u212-jdk-stretch</image>
                    </from>
                    <to>
                        <!--镜像名称和tag，使用了mvn内置变量${project.version}，表示当前工程的version-->
                        <image>bolingcavalry/hellojib:${project.version}</image>
                    </to>
                    <!--容器相关的属性-->
                    <container>
                        <!--jvm内存参数-->
                        <jvmFlags>
                            <jvmFlag>-Xms4g</jvmFlag>
                            <jvmFlag>-Xmx4g</jvmFlag>
                        </jvmFlags>
                        <!--要暴露的端口-->
                        <ports>
                            <port>8080</port>
                        </ports>
                        <!--使用该参数将镜像的创建时间与系统时间对其-->
                        <useCurrentTimestamp>true</useCurrentTimestamp>
                    </container>
                </configuration>
            </plugin>

```

修改保存后再次构建，此时新的镜像的创建时间已经正确，显示15 seconds ago：

```shell
[root@maven hellojib]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
bolingcavalry/hellojib   0.0.1-SNAPSHOT      ee5ba19a8016        23 seconds ago      505 MB
<none>                   <none>              ef96fdd4473a        49 years ago        505 MB
1234
```

### 小结三：多次构建后，积累的无用镜像

如下所示，构建多次后，本地会遗留多个名为<none>，tag也是<none>的镜像：

```shell
[root@maven hellojib]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED              SIZE
bolingcavalry/hellojib   0.0.1-SNAPSHOT      a9fd91d8ad8c        17 seconds ago       505 MB
<none>                   <none>              a0cadeb9febd        About a minute ago   505 MB
<none>                   <none>              ee5ba19a8016        2 minutes ago        505 MB
<none>                   <none>              ef96fdd4473a        49 years ago         505 MB
123456
```

这些都是上一次构建的结果，在经历了新一轮的构建后，其镜像名和tag被新镜像所有，所以自身只能显示名为<none>，tag也是<none>，清理这些镜像的命令是docker image prune，然后根据提示输入"y"，镜像即可被清理：

```shell
[root@maven hellojib]# docker image prune
WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
Deleted Images:
deleted: sha256:7aa104e20b8a08bac3255f2627ac90f10021c6630370ce7a84ba33f89404b153
deleted: sha256:7dd7376ae00c2df0411bac1eded4b3c79dd1528f5711057fe11a4f4121504486
deleted: sha256:e71ced47e80a7fccfea1710f1e5a257d4e16fc3e96b05616007e15829e71a7b2
deleted: sha256:55bed58453479c2accfc08fabc929aece7d324af0df94335dd46333db9da1d23
deleted: sha256:ef96fdd4473a7ca9d39a50e0feae50131de083cee4f11060ad8bee1bc853b2b5

Total reclaimed space: 0 B
[root@maven hellojib]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED              SIZE
bolingcavalry/hellojib   0.0.1-SNAPSHOT      3afd4165b6b6        About a minute ago   505 MB
1234567891011121314
```

### 小结四：提升构建速度

在使用命令mvn compile jib:dockerBuild构建本地镜像时，每次构建的过程中都会提示以下信息：

```shell
[INFO] Containerizing application to Docker daemon as bolingcavalry/hellojib:0.0.1-SNAPSHOT...
[INFO] The base image requires auth. Trying again for openjdk:8-jdk-stretch...
[INFO] Executing tasks:
[INFO] [=========                     ] 30.0% complete
[INFO] > pulling base image manifest
12345
```

每次构建都会显示上面的内容，也就是说每次都去远程拉取base镜像的manifest(pulling base image manifest)，这部分时间导致整体构架时间变长，以下是构建结果，可见用了10秒：

```shell
[INFO] Built image to Docker daemon as bolingcavalry/hellojib:0.0.1-SNAPSHOT
[INFO] Executing tasks:
[INFO] [==============================] 100.0% complete
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  10.722 s
[INFO] Finished at: 2019-09-01T08:55:09+08:00
[INFO] ------------------------------------------------------------------------
12345678910
```

首先想到的是执行命令docker pull openjdk:8-jdk-stretch将base镜像下载到本地仓库，再尝试构建，遗憾的是jib依旧会远程获取base镜像的manifest，还是很慢；

如果能避免远程拉取base镜像的manifest，镜像构建速度应该会快一些；基于此推论，优化构建速度的步骤如下：

1. 在本机创建registry(docker镜像仓库服务)；
2. 将base镜像openjdk:8-jdk-stretch放入本机registry；
3. 修改pom.xml中base镜像的配置，改为本机registry的镜像；
4. 如此一来，每次都会从本机registry取得base镜像的manifest，不走远程请求响应，构建时间会有提升；

接下按照上述步骤进行操作：

1. 确认当前电脑的IP地址，我这里是192.168.121.131；
2. 设置本地docker服务支持http：修改docker配置文件：/etc/docker/daemon.json，在json中增加内容"insecure-registries": [“192.168.121.131:5000”]
3. 重启docker使配置生效：

```shell
systemctl restart docker
```

1. 在本地创建一个镜像仓库服务：

```shell
docker run --name docker-registry -d -p 5000:5000 registry
```

1. 查看本地镜像openjdk:8-jdk-stretch的ID为08ded5f856cc；
2. 用tag命令将本地镜像openjdk:8-jdk-stretch改名，命令如下所示，"192.168.121.131"是当前电脑的IP地址：

```shell
docker tag 08ded5f856cc 192.168.121.131:5000/openjdk:8-jdk-stretch
```

1. 再次查看镜像，发现多了个192.168.121.131:5000/openjdk:8u212-jdk-stretch：

```shell
[root@maven hellojib]# docker tag 08ded5f856cc 192.168.121.131:5000/openjdk:8-jdk-stretch
[root@maven hellojib]# docker images
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
bolingcavalry/hellojib         0.0.1-SNAPSHOT      6601ef5a767d        3 minutes ago       505 MB
192.168.121.131:5000/openjdk   8-jdk-stretch       08ded5f856cc        2 weeks ago         488 MB
docker.io/openjdk              8-jdk-stretch       08ded5f856cc        2 weeks ago         488 MB
123456
```

1. 将192.168.121.131:5000/openjdk:8-jdk-stretch推送到远程仓库，命令如下所示，由于镜像名前缀是192.168.121.131:5000，镜像会被推送到我们刚刚创建的registry：

```shell
docker push 192.168.121.131:5000/openjdk:8-jdk-stretch
```

1. 修改java工程的pom.xml，将base镜像由之前的openjdk:8-jdk-stretch改为192.168.121.131:5000/openjdk:8-jdk-stretch
2. 修改java工程的pom.xml，增加allowInsecureRegistries，使jib插件支持http协议连接镜像仓库(安全起见，默认是关闭的)，整个插件的配置信息如下：

```xml
<plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <!--使用jib插件-->
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>1.3.0</version>
                <configuration>
                    <!--from节点用来设置镜像的基础镜像，相当于Docerkfile中的FROM关键字-->
                    <from>
                        <!--使用openjdk官方镜像，tag是8-jdk-stretch，表示镜像的操作系统是debian9,装好了jdk8-->
                        <image>192.168.121.131:5000/openjdk:8-jdk-stretch</image>
                    </from>
                    <to>
                        <!--镜像名称和tag，使用了mvn内置变量${project.version}，表示当前工程的version-->
                        <image>bolingcavalry/hellojib:${project.version}</image>
                    </to>
                    <!--容器相关的属性-->
                    <container>
                        <!--jvm内存参数-->
                        <jvmFlags>
                            <jvmFlag>-Xms4g</jvmFlag>
                            <jvmFlag>-Xmx4g</jvmFlag>
                        </jvmFlags>
                        <!--要暴露的端口-->
                        <ports>
                            <port>8080</port>
                        </ports>
                        <useCurrentTimestamp>true</useCurrentTimestamp>
                    </container>
                    <allowInsecureRegistries>true</allowInsecureRegistries>
                </configuration>
            </plugin>
        </plugins>

```

1. 再次执行命令mvn compile jib:dockerBuild，如下所示，时间从之前的10秒缩减到3.9秒：

```shell
[INFO] Built image to Docker daemon as bolingcavalry/hellojib:0.0.1-SNAPSHOT
[INFO] Executing tasks:
[INFO] [==============================] 100.0% complete
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  3.949 s
[INFO] Finished at: 2019-09-01T10:51:50+08:00
[INFO] ------------------------------------------------------------------------
12345678910
```

以上就是通过registry服务提升jib构建速度的方法，在多人开发的时候，registry可以配置为一个公共的，多人都可使用，毕竟pom.xml代码存在公共代码仓库，每个人都去修改成自己的registry的地址是不合适的，一旦提交上去会影响其他人的使用，我们这里的做法是将registry的地址写成host，本地维护好host和IP的映射就可以了。
使用本地registry服务，除了提速，在服务器无法连接中央仓库取得openjdk:8-jdk-stretch的manifest时，这种方式能保证构建依旧能够成功；

### 小结五：将jib与mvn构建的生命周期绑定

1. 前面的实战中构建命令是mvn compile jib:dockerBuild，实际上可以做到仅用mvn compile就完成镜像构建，这是maven插件的通用特性；
2. 修改pom.xml增加executions节点，里面设置compile触发jib:dockerBuild，整个插件的内容如下所示：

```xml
<plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>1.3.0</version>
                <configuration>
                    <!--from节点用来设置镜像的基础镜像，相当于Docerkfile中的FROM关键字-->
                    <from>
                        <!--使用openjdk官方镜像，tag是8-jdk-stretch，表示镜像的操作系统是debian9,装好了jdk8-->
                        <image>192.168.121.131:5000/openjdk:8-jdk-stretch</image>
                    </from>
                    <to>
                        <!--镜像名称和tag，使用了mvn内置变量${project.version}，表示当前工程的version-->
                        <image>bolingcavalry/hellojib:${project.version}</image>
                    </to>
                    <!--容器相关的属性-->
                    <container>
                        <!--jvm内存参数-->
                        <jvmFlags>
                            <jvmFlag>-Xms4g</jvmFlag>
                            <jvmFlag>-Xmx4g</jvmFlag>
                        </jvmFlags>
                        <!--要暴露的端口-->
                        <ports>
                            <port>8080</port>
                        </ports>
                        <useCurrentTimestamp>true</useCurrentTimestamp>
                    </container>
                    <allowInsecureRegistries>true</allowInsecureRegistries>
                </configuration>
                <executions>
                  <execution>
                    <phase>compile</phase>
                    <goals>
                      <goal>dockerBuild</goal>
                    </goals>
                  </execution>
                </executions>
            </plugin>
```

1. 执行命令mvn compile -DskipTests，如下所示，可以成功构建镜像，与前面的命令结果一致：

```shell
[root@maven hellojib]# mvn compile -DskipTests
[INFO] Scanning for projects...
[INFO] 
[INFO] ---------------------< com.bolingcavalry:hellojib >---------------------
[INFO] Building hellojib 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-resources-plugin:3.1.0:resources (default-resources) @ hellojib ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ hellojib ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- jib-maven-plugin:1.3.0:dockerBuild (default) @ hellojib ---
[WARNING] Setting image creation time to current time; your image may not be reproducible.
[INFO] 
[INFO] Containerizing application to Docker daemon as bolingcavalry/hellojib:0.0.1-SNAPSHOT...
[INFO] 
[INFO] Container entrypoint set to [java, -Xms4g, -Xmx4g, -cp, /app/resources:/app/classes:/app/libs/*, com.bolingcavalry.hellojib.HellojibApplication]
[INFO] 
[INFO] Built image to Docker daemon as bolingcavalry/hellojib:0.0.1-SNAPSHOT
[INFO] Executing tasks:
[INFO] [==============================] 100.0% complete
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  3.721 s
[INFO] Finished at: 2019-09-01T11:43:23+08:00
[INFO] ------------------------------------------------------------------------
[root@maven hellojib]# docker images
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
bolingcavalry/hellojib         0.0.1-SNAPSHOT      5e3f62d13a73        35 seconds ago      505 MB
192.168.121.131:5000/openjdk   8-jdk-stretch       08ded5f856cc        2 weeks ago         488 MB
docker.io/openjdk              8-jdk-stretch       08ded5f856cc        2 weeks ago         488 MB
docker.io/registry             latest              f32a97de94e1        5 months ago        25.8 MB
1234567891011121314151617181920212223242526272829303132333435363738
```

### 小结六：父子结构的maven工程如何构建

1. 假设当前maven工程是父子结构的，有两个子工程A和B，其中A是二方库，提供一个jar包，里面是接口类和Bean类，B是springboot应用，并且B的源码中用到了A提供的接口和Bean；
2. 上述父子结构的maven工程是常见的工程结构，此时如果要将B构建成Docker镜像，在B的目录下执行mvn compile jib:dockerBuild显然是不行的，因为没有编译构建A，会导致B的编译失败；
3. 此时最好的做法就是将jib与mvn构建的生命周期绑定，修改B的pom.xml文件，加入executions节点；
4. 在父工程目录下执行mvn compile，此时maven会先编译构建整个工程，然后再将B工程的构建结果制作成镜像；  

### 小结七：jib推送到远程仓库权限的问题

##### 准备工作

在39.100.127.235容器镜像服务中创建镜像仓库，开放http端口8009。

![image-20210105105438984](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210105105438984.png)

##### 一、设置jdk镜像

在管理镜像中参考官方给的demo进行设置即可,这里把jdk镜像上传到自己构建的镜像仓库

![image-20210105105619030](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210105105619030.png)

##### 二、设置pom.xml

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <configuration>
                    <from>
                        <image>39.100.127.235:8009/openjdk:8-jdk-alpine</image>
                    </from>
                    <to>
                        <image>39.100.127.235:8009/individualgrid</image>
                        <tags>
                            <tag>0.01</tag>
                        </tags>
                    </to>
                    <container>
                        <entrypoint>
                            <shell>bash</shell>
                            <option>-c</option>
                            <arg>chmod +x /entrypoint.sh &amp;&amp; sync &amp;&amp; /entrypoint.sh</arg>
                        </entrypoint>
                        <ports>
                            <port>8080</port>
                        </ports>
                        <useCurrentTimestamp>true</useCurrentTimestamp>
                        <args>
                            <arg>--spring.profiles.active=prod</arg>
                        </args>
                    </container>
                    <!-- 允许仓库配置的http的仓库私服地址,这个不配置会报错,同时推送到远程仓库时需要加命令行参数 
                    -DsendCredentialsOverHttp=true 不加也会报权限问题 -->
                    <allowInsecureRegistries>true</allowInsecureRegistries>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

方法一：配置`maven settings.xml`文件（推荐）

```xml
    <pluginGroups>
        <!-- pluginGroup | Specifies a further group identifier to use for plugin
            lookup. <pluginGroup>com.your.plugins</pluginGroup> -->
        <pluginGroup>com.google.cloud.tools</pluginGroup>
    </pluginGroups>
    <!--对应容器仓库权限的账号密码-->
    <servers>
        <server>
            <id>39.100.127.235:8009</id>
            <username>grid</username>
            <password>grid@123</password>
        </server>
    </servers>
```

方法二：在pom.xml中添加认证信息（不推荐）

```xml
                    <from>
                        <image>registry.cn-hangzhou.aliyuncs.com/yhhu/jdk8</image>
                        <auth>
                            <username>my_username</username>
                            <password>my_password</password>
                        </auth>
                    </from>
                    <to>
                        <image>registry.cn-hangzhou.aliyuncs.com/yhhu/ejile</image>
                        <tags>
                            <tag>0.01</tag>
                        </tags>
                        <auth>
                            <username>my_username</username>
                            <password>my_password</password>
                        </auth>
                    </to>
```

##### 三、构建并提交镜像

```sh
mvn compile jib:build -DsendCredentialsOverHttp=true
```

### 扩展使用

扩展选项提供额外的配置选项为定制镜像生成。

| 属性                      | 类型        | 默认值          | 描述                                                         |
| :------------------------ | :---------- | :-------------- | :----------------------------------------------------------- |
| `to`                      | `to`        | Required        | 配置目标镜像以构建应用程序。                                 |
| `from`                    | `from`      | 查看`from`      | 配置基础镜像以构建应用程序的顶部。                           |
| `container`               | `container` | 查看`container` | 配置从镜像中运行的容器。                                     |
| `allowInsecureRegistries` | boolean     | `false`         | 如果设置为`true`，Jib忽略HTTPS证书错误，并可能退回HTTP作为最后的手段。强烈建议将这个参数设置为`false`，因为HTTP通信是未加密的，并且对于网络上的其他人来说是可见的，并且不安全的HTTPS并不比普通的HTTP好。如果使用自签名证书访问注册表，则将证书添加到Java运行时的可信密钥可能是启用此选项的另一种选择。 |
| `skip`                    | boolean     | `false`         | 如果设置为`true`，则跳过Jib执行（对于多模块项目是有用的）。这也可以通过`-Djib.skip`跳过命令行选项来指定。 |
| `useOnlyProjectCache`     | boolean     | `false`         | 如果设置为`true`，Jib不会在不同的Maven项目之间共享缓存。     |

`from`是具有以下属性的对象：

| 属性         | 类型   | 默认值                   | 描述                                                         |
| :----------- | :----- | :----------------------- | :----------------------------------------------------------- |
| `image`      | string | `gcr.io/distroless/java` | 基础的镜像参考。                                             |
| `auth`       | `auth` | None                     | 直接指定凭证。                                               |
| `credHelper` | string | None                     | 证书助手的后缀，它可以对基础镜像进行身份验证（遵循`docker-credential-`）。 |

`to`是具有以下属性的对象：

| 属性         | 类型   | 默认值   | 描述                                                         |
| :----------- | :----- | :------- | :----------------------------------------------------------- |
| `image`      | string | Required | 目标镜像的参考。这也可以通过`-Dimage`命令行选项来指定。      |
| `auth`       | `auth` | None     | 直接指定凭证。                                               |
| `credHelper` | string | None     | 证书助手的后缀，它可以对基础镜像进行身份验证（遵循`docker-credential-`）。 |
| `tags`       | list   | None     | 额外的标签推向                                               |

`auth`是一个具有以下属性的对象（请参阅使用特定凭据）：

| 属性       | 类型   | 描述   |
| :--------- | :----- | :----- |
| `username` | string | 用户名 |
| `password` | string | 密码   |

`container`是具有以下属性的对象：

| 属性                  | 类型    | 默认值    | 描述                                                         |
| :-------------------- | :------ | :-------- | :----------------------------------------------------------- |
| `appRoot`             | string  | `/app`    | 容器上放置应用程序内容的根目录。                             |
| `args`                | list    | None      | 默认的主方法参数来运行应用程序。                             |
| `entrypoint`          | list    | None      | 启动容器的命令（类似于Docker的入口点指令）。如果设置，则忽略`jvmFlags`和`mainClass`。 |
| `environment`         | map     | None      | 键值对，用于设置容器上的环境变量（类似于Docker的Env指令）。  |
| `format`              | string  | `Docker`  | 使用`OCI`构建OCI容器映像                                     |
| `jvmFlags`            | list    | None      | 运行应用程序时要传递给JVM的附加标志。                        |
| `labels`              | map     | None      | 用于应用镜像元数据的键值对（类似于Docker的标签指令）。       |
| `mainClass`           | string  | Inferred* | 主要类从应用程序启动。                                       |
| `ports`               | list    | None      | 容器在运行时暴露的端口（类似于Docker的公开指令）。           |
| `useCurrentTimestamp` | boolean | `false`   | 默认情况下，Jib擦除所有时间戳以保证再现性。如果这个参数设置为`true`，Jib将把镜像的创建时间戳设置为构建时间，这将牺牲可再现性，以便能够容易地判断镜像何时创建。 |

您还可以使用`connection/read`系统属性为注册表交互配置HTTP连接/读取超时，该属性通过命令行以毫秒为单位配置（缺省值是`20000`；您还可以将其设置为`0`用于无限超时）：

```
mvn compile jib:build -Djib.httpTimeout=3000
```

#### 例子

在这种配置中，镜像：

- 是从`openjdk:alpine`的基础上建造的：(pulled from Docker Hub)
- 被推到`localhost:5000/my-image:built-with-jib`：`localhost:5000/my-image:tag2`，`localhost:5000/my-image:latest`
- 通过调用运行`java -Xms512m -Xdebug -Xmy:flag=jib-rules -cp app/libs/*:app/resources:app/classes mypackage.MyApp some args`
- 暴露端口1000为TCP（默认），并且端口2000、2001、2002和2003为UDP，
- 有两个标签(key1:value1 and key2:value2)
- 构建为OCI格式

```
<configuration>
  <from>
    <image>openjdk:alpine</image>
  </from>
  <to>
    <image>localhost:5000/my-image:built-with-jib</image>
    <credHelper>osxkeychain</credHelper>
    <tags>
      <tag>tag2</tag>
      <tag>latest</tag>
    </tags>
  </to>
  <container>
    <jvmFlags>
      <jvmFlag>-Xms512m</jvmFlag>
      <jvmFlag>-Xdebug</jvmFlag>
      <jvmFlag>-Xmy:flag=jib-rules</jvmFlag>
    </jvmFlags>
    <mainClass>mypackage.MyApp</mainClass>
    <args>
      <arg>some</arg>
      <arg>args</arg>
    </args>
    <ports>
      <port>1000</port>
      <port>2000-2003/udp</port>
    </ports>
    <labels>
      <key1>value1</key1>
      <key2>value2</key2>
    </labels>
    <format>OCI</format>
  </container>
</configuration>
```

#### 在镜像中添加任意文件

*注意：这是一个孵化的特点，并可能在未来改变。*

可以将任意的、非类路径文件添加到`src/main/jib`目录中，从而将它们添加到镜像中。这将将jib文件夹中的所有文件复制到镜像的根目录，并保持相同的结构（例如，如果文本文件在`src/main/jib/dir/hello.txt`,然后你的镜像就会包含`/dir/hello.txt`Jib建成后。

您可以使用`pom.xml`中的`extraDirectory`参数配置一个不同的目录：

```
<configuration>
  ...
  <!-- Copies files from 'src/main/custom-extra-dir' instead of 'src/main/jib' -->
  <extraDirectory>${project.basedir}/src/main/custom-extra-dir</extraDirectory>
  ...
</configuration>
```

#### 认证方法

来自私人注册中心的推/拉需要授权凭证。这些可以使用Docker证书帮助器或在Maven设置中定义。如果不显式地定义凭据，Jib将尝试使用Docker配置中定义的凭据或推断公共凭据助手。

#### 使用Docker凭证帮助器

Docker证书助手是CLI工具，它处理各种注册表的身份验证。

一些常见的证书帮助者包括：

- Google Container Registry: [docker-credential-gcr](https://cloud.google.com/container-registry/docs/advanced-authentication#docker_credential_helper)
- AWS Elastic Container Registry: [docker-credential-ecr-login](https://github.com/awslabs/amazon-ecr-credential-helper)
- Docker Hub Registry: [docker-credential-*](https://github.com/docker/docker-credential-helpers)

通过将证书指定为它们各自的镜像的帮助器来配置证书助手。

示例配置：

```
<configuration>
  ...
  <from>
    <image>aws_account_id.dkr.ecr.region.amazonaws.com/my-base-image</image>
    <credHelper>ecr-login</credHelper>
  </from>
  <to>
    <image>gcr.io/my-gcp-project/my-app</image>
    <credHelper>gcr</credHelper>
  </to>
  ...
</configuration>
```

#### 使用特定凭证

您可以在`<auth>`参数为`from` and/or `to`镜像。在下面的示例中，`to`凭据检索凭据`REGISTRY_USERNAME`和`REGISTRY_PASSWORD`环境变量。

```
<configuration>
  ...
  <from>
    <image>aws_account_id.dkr.ecr.region.amazonaws.com/my-base-image</image>
    <auth>
      <username>my_username</username>
      <password>my_password</password>
    </auth>
  </from>
  <to>
    <image>gcr.io/my-gcp-project/my-app</image>
    <auth>
      <username>${env.REGISTRY_USERNAME}</username>
      <password>${env.REGISTRY_PASSWORD}</password>
    </auth>
  </to>
  ...
</configuration>
```

或者，可以使用下面的系统属性通过命令行指定凭据。

| 属性                       | 描述                     |
| :------------------------- | :----------------------- |
| `-Djib.from.auth.username` | 基本镜像注册表的用户名。 |
| `-Djib.from.auth.password` | 基本镜像注册表的密码。   |
| `-Djib.to.auth.username`   | 目标镜像注册表的用户名。 |
| `-Djib.to.auth.password`   | 目标镜像注册表的密码。   |

例如：`mvn compile jib:build -Djib.to.auth.username=user -Djib.to.auth.password=pass`

**注意：这种身份验证方法只能作为最后的手段，因为用纯文本显示密码是不安全的。**

#### 使用Maven设置

注册表凭据可以添加到您的Maven设置。如果在任何指定的Docker凭证帮助器中找不到凭据，则将使用这些凭据。

如果您考虑在Maven中设置凭据，我们强烈建议使用Maven密码加密。

示例 `settings.xml`:

```
<settings>
  ...
  <servers>
    ...
    <server>
      <id>MY_REGISTRY</id>
      <username>MY_USERNAME</username>
      <password>{MY_SECRET}</password>
    </server>
  </servers>
</settings>
```

- id字段应该是这些凭据所需的注册表服务器。
- 我们不建议在`settings.xml`中添加原始密码。



以上就是我在近期使用Jib插件时遇到的问题小结，希望这些小技巧可以给您提供一些参考，助您解决问题；

