

#                    Maven私服Nexus3.x环境构建操作记录

#### **一、Maven介绍**

Apache Maven是一个创新的软件项目管理和综合工具。
Maven提供了一个基于项目对象模型（POM）文件的新概念来管理项目的构建，可以从一个中心资料片管理项目构建，报告和文件。
Maven最强大的功能就是能够自动下载项目依赖库。
Maven提供了开发人员构建一个完整的生命周期框架。开发团队可以自动完成项目的基础工具建设，Maven使用标准的目录结构和默认构建生命周期。
在多个开发团队环境时，Maven可以设置按标准在非常短的时间里完成配置工作。由于大部分项目的设置都很简单，并且可重复使用，Maven让开发人员的工作更轻松，同时创建报表，检查，构建和测试自动化设置。
Maven项目的结构和内容在一个XML文件中声明，pom.xml 项目对象模型（POM），这是整个Maven系统的基本单元。

Maven提供了开发人员的方式来管理：
1）Builds
2）Documentation
3）Reporting
4）Dependencies
5）SCMs
6）Releases
7）Distribution
8）mailing list
概括地说，Maven简化和标准化项目建设过程。处理编译，分配，文档，团队协作和其他任务的无缝连接。 
Maven增加可重用性并负责建立相关的任务。
Maven最初设计，是以简化Jakarta Turbine项目的建设。在几个项目，每个项目包含了不同的Ant构建文件。 JAR检查到CVS。
Apache组织开发Maven可以建立多个项目，发布项目信息，项目部署，在几个项目中JAR文件提供团队合作和帮助。

Maven主要目标是提供给开发人员：
1）项目是可重复使用，易维护，更容易理解的一个综合模型。
2）插件或交互的工具，这种声明性的模式。

#### **二、私服介绍**

私服是指私有服务器，是架设在局域网的一种特殊的远程仓库，目的是代理远程仓库及部署第三方构建。有了私服之后，当 Maven 需要下载构件时，直接请求私服，私服上存在则下载到本地仓库；否则，私服请求外部的远程仓库，将构件下载到私服，再提供给本地仓库下载。

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161220162726057-620806393.png)

#### **三、Nexus介绍**

Nexus是一个强大的Maven仓库管理器，它极大地简化了本地内部仓库的维护和外部仓库的访问。
如果使用了公共的Maven仓库服务器，可以从Maven中央仓库下载所需要的构件（Artifact），但这通常不是一个好的做法。
正常做法是在本地架设一个Maven仓库服务器，即利用Nexus私服可以只在一个地方就能够完全控制访问和部署在你所维护仓库中的每个Artifact。
Nexus在代理远程仓库的同时维护本地仓库，以降低中央仓库的负荷,节省外网带宽和时间，Nexus私服就可以满足这样的需要。
Nexus是一套“开箱即用”的系统不需要数据库，它使用文件系统加Lucene来组织数据。 
Nexus使用ExtJS来开发界面，利用Restlet来提供完整的REST APIs，通过m2eclipse与Eclipse集成使用。 
Nexus支持WebDAV与LDAP安全身份认证。 
Nexus还提供了强大的仓库管理功能，构件搜索功能，它基于REST，友好的UI是一个extjs的REST客户端，它占用较少的内存，基于简单文件系统而非数据库。

为什么要构建Nexus私服？
如果没有Nexus私服，我们所需的所有构件都需要通过maven的中央仓库和第三方的Maven仓库下载到本地，而一个团队中的所有人都重复的从maven仓库下载构件无疑加大了仓库的负载和浪费了外网带宽，如果网速慢的话，还会影响项目的进程。很多情况下项目的开发都是在内网进行的，连接不到maven仓库怎么办呢？开发的公共构件怎么让其它项目使用？这个时候我们不得不为自己的团队搭建属于自己的maven私服，这样既节省了网络带宽也会加速项目搭建的进程，当然前提条件就是你的私服中拥有项目所需的所有构件。

总之，在本地构建nexus私服的好处有：
1）加速构建；
2）节省带宽；
3）节省中央maven仓库的带宽；
4）稳定（应付一旦中央服务器出问题的情况）；
5）控制和审计；
6）能够部署第三方构件；
7）可以建立本地内部仓库；
8）可以建立公共仓库
这些优点使得Nexus日趋成为最流行的Maven仓库管理器。

#### **四、Maven的安装**

下载地址：http://maven.apache.org/download.cgi
提前在服务器上安装jdk环境（参考：[Centos中yum方式安装java](http://www.cnblogs.com/kevingrace/p/5870814.html)）
[root@master-node ~]# cd /usr/local/src/
[root@master-node src]# wget http://mirrors.hust.edu.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
[root@master-node src]# tar -zvxf apache-maven-3.3.9-bin.tar.gz 
[root@master-node src]# mv apache-maven-3.3.9 /usr/local/maven

接着配置系统环境变量，在/etc/profile文件底部添加如下内容：
[root@master-node src]# java -version
openjdk version "1.8.0_111"
OpenJDK Runtime Environment (build 1.8.0_111-b15)
OpenJDK 64-Bit Server VM (build 25.111-b15, mixed mode)
[root@master-node src]# vim /etc/profile
.....
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk                 //java的环境变量设置
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin

export MAVEN_HOME=/usr/local/maven                         //maven的环境变量设置
export PATH=$PATH:$MAVEN_HOME/bin
[root@master-node src]# source /etc/profile

最后验证是否安装成功，出现如下信息，说明安装成功
[root@master-node src]# mvn --version          # 最好按照java jdk
Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
Maven home: /usr/local/maven
Java version: 1.8.0_111, vendor: Oracle Corporation
Java home: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-2.b15.el7_3.x86_64/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-327.el7.x86_64", arch: "amd64", family: "unix"

#### **五、Nexus安装**

Nexus的安装有两种实现方式：
1）war包安装方式
下载地址：https://sonatype-download.global.ssl.fastly.net/nexus/oss/nexus-2.14.2-01.war
直接将war包放在tomcat的根目录下，启动tomcat就可以用了

2）源码安装方式（之前在用的是2.14.4版本，这里是新版本）
下载地址：https://www.sonatype.com/download-oss-sonatype      （云盘下载：http://pan.baidu.com/s/1miKFm5a）
[root@master-node ~]# cd /usr/local/src/
[root@master-node src]# wget https://sonatype-download.global.ssl.fastly.net/nexus/3/nexus-3.2.0-01-unix.tar.gz
[root@master-node src]# tar -zvxf nexus-3.2.0-01-unix.tar.gz 
[root@master-node src]# mv nexus-3.2.0-01 /usr/local/nexus

启动nexus（默认端口是8081）
[root@master-node src]# /usr/local/nexus/bin/nexus 
WARNING: ************************************************************
WARNING: Detected execution as "root" user. This is NOT recommended!
WARNING: ************************************************************
Usage: /usr/local/nexus/bin/nexus {start|stop|run|run-redirect|status|restart|force-reload}
[root@master-node src]# /usr/local/nexus/bin/nexus start
WARNING: ************************************************************
WARNING: Detected execution as "root" user. This is NOT recommended! 
WARNING: ************************************************************
Starting nexus
上面在启动过程中出现告警：不推荐使用root用户启动。这个告警不影响nexus的正常访问和使用。
去掉上面WARNING的办法：
[root@master-node src]# vim /etc/profile
......
export RUN_AS_USER=root
[root@master-node src]# source /etc/profile
[root@master-node src]#                               //nexus服务启动成功后，需要稍等一段时间，8081端口才起来
COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME
java 1486 root 859u IPv4 23504303 0t0 TCP *:tproxy (LISTEN)

在部署机上的iptables里打开8081端口
[root@master-node src]# vim /etc/sysconfig/iptables
....
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8081 -j ACCEPT
[root@master-node src]# /etc/init.d/iptables restart

访问nexus，即[http://39.100.127.235:8081](http://39.100.127.235:8081/) （如果出现404，就访问http://39.100.127.235:8081/nexus）

出现上述页面，说明配置nexus成功！
点击右上角“Log in”，
输入默认用户名(admin)和默认密码（admin123）登录

可以点击上面的“设置”图标，在“设置”里可以添加用户、角色，对接LDAP等的设置，如下：

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161220154712776-67407111.png)

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161220154756276-1502291718.png)

可以在“管理”里查看nexus的系统信息

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161220160738011-558914286.png)

#### **六、Nexus说明**

1.component name的一些说明： 
  1）maven-central：maven中央库，默认从https://repo1.maven.org/maven2/拉取jar 
  2）maven-releases：私库发行版jar 
  3）maven-snapshots：私库快照（调试版本）jar 
  4）maven-public：仓库分组，把上面三个仓库组合在一起对外提供服务，在本地maven基础配置settings.xml中使用。
2.Nexus默认的仓库类型有以下四种：
  1）group(仓库组类型)：又叫组仓库，用于方便开发人员自己设定的仓库；
  2）hosted(宿主类型)：内部项目的发布仓库（内部开发人员，发布上去存放的仓库）；
  3）proxy(代理类型)：从远程中央仓库中寻找数据的仓库（可以点击对应的仓库的Configuration页签下Remote Storage Location属性的值即被代理的远程仓库的路径）；
  4）virtual(虚拟类型)：虚拟仓库（这个基本用不到，重点关注上面三个仓库的使用）；
3.Policy(策略):表示该仓库为发布(Release)版本仓库还是快照(Snapshot)版本仓库；
4.Public Repositories下的仓库 
  1）3rd party: 无法从公共仓库获得的第三方发布版本的构件仓库，即第三方依赖的仓库，这个数据通常是由内部人员自行下载之后发布上去；
  2）Apache Snapshots: 用了代理ApacheMaven仓库快照版本的构件仓库 
  3）Central: 用来代理maven中央仓库中发布版本构件的仓库 
  4）Central M1 shadow: 用于提供中央仓库中M1格式的发布版本的构件镜像仓库 
  5）Codehaus Snapshots: 用来代理CodehausMaven 仓库的快照版本构件的仓库 
  6）Releases: 内部的模块中release模块的发布仓库，用来部署管理内部的发布版本构件的宿主类型仓库；release是发布版本；
  7）Snapshots:发布内部的SNAPSHOT模块的仓库，用来部署管理内部的快照版本构件的宿主类型仓库；snapshots是快照版本，也就是不稳定版本
所以自定义构建的仓库组代理仓库的顺序为：Releases，Snapshots，3rd party，Central。也可以使用oschina放到Central前面，下载包会更快。
5.Nexus默认的端口是8081，可以在etc/nexus-default.properties配置中修改。
6.Nexus默认的用户名密码是admin/admin123
7.当遇到奇怪问题时，重启nexus，重启后web界面要1分钟左右后才能访问。
8.Nexus的工作目录是sonatype-work（路径一般在nexus同级目录下）
[root@master-node local]# pwd
/usr/local
[root@master-node local]# ls nexus/
bin deploy etc lib LICENSE.txt NOTICE.txt public system
[root@master-node local]# ls sonatype-work/
nexus3
[root@master-node local]# ls sonatype-work/nexus3/
backup blobs cache db elasticsearch etc generated-bundles health-check instances keystores lock log orient port tmp

**Nexus仓库分类的概念**
1）Maven可直接从宿主仓库下载构件,也可以从代理仓库下载构件,而代理仓库间接的从远程仓库下载并缓存构件 
2）为了方便,Maven可以从仓库组下载构件,而仓库组并没有时间的内容(下图中用虚线表示,它会转向包含的宿主仓库或者代理仓库获得实际构件的内容).

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161220164452667-1588016524.png)

 

**Nexus的web界面功能介绍**

**1.Browse Server Content**

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221105344026-1095529496.png)

1.1  Search
这个就是类似Maven仓库上的搜索功能，就是从私服上查找是否有哪些包。
注意：
1）在Search这级是支持模糊搜索的，如图所示：

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221105547167-2017264888.png)

2）如果进入具体的目录，好像不支持模糊搜索，如图所示：

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221105634245-19314456.png)

1.2  Browse

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221105808261-1531078817.png)

1）Assets
这是能看到所有的资源，包含Jar，已经对Jar的一些描述信息。
2）Components
这里只能看到Jar包。

**2.Server Adminstration And configuration**

看到这个选项的前提是要进行登录的，如上面已经介绍登陆方法，右上角点击“Sign In”的登录按钮，输入admin/admin123,登录成功之后，即可看到此功能，如图所示：

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221110207479-318443915.png)

2.1 Blob Stores
文件存储的地方，创建一个目录的话，对应文件系统的一个目录，如图所示：

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221110317932-1418604671.png)

2.2 Repositories

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221110524073-153249455.png)

1）Proxy
这里就是代理的意思，代理中央Maven仓库，当PC访问中央库的时候，先通过Proxy下载到Nexus仓库，然后再从Nexus仓库下载到PC本地。
这样的优势只要其中一个人从中央库下来了，以后大家都是从Nexus私服上进行下来，私服一般部署在内网，这样大大节约的宽带。
创建Proxy的具体步骤
1--点击“Create Repositories”按钮

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221110640089-1389328954.png)

2--选择要创建的类型

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221110758542-1042948386.png)

3--填写详细信息
Name：就是为代理起个名字
Remote Storage: 代理的地址，Maven的地址为: https://repo1.maven.org/maven2/阿里云的地址为：https://maven.aliyun.com/repository/public
Blob Store: 选择代理下载包的存放路径

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221111233651-1321037653.png)

2）Hosted
Hosted是宿主机的意思，就是怎么把第三方的Jar放到私服上。
Hosted有三种方式，Releases、SNAPSHOT、Mixed
Releases: 一般是已经发布的Jar包
Snapshot: 未发布的版本
Mixed：混合的
Hosted的创建和Proxy是一致的，具体步骤和上面基本一致。如下：

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221111325104-723208432.png)

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221111344573-1949446719.png)

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221111419870-2005652543.png)

**注意事项：**
Deployment Pollcy: 需要把策略改成“Allow redeploy”。

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221111455589-1303191395.png)

3）Group
能把两个仓库合成一个仓库来使用，目前没使用过，所以没做详细的研究。

2.3 Security
这里主要是用户、角色、权限的配置（上面已经提到了在这里添加用户和角色等）

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221111747745-1991046629.png)

2.4 Support
包含日志及数据分析。

![img](https://images2015.cnblogs.com/blog/907596/201612/907596-20161221111920261-269936853.png)

2.5 System
主要是邮件服务器，调度的设置地方
这部分主要讲怎么和Maven做集成,集成的方式主要分以下种情况：代理中央仓库、Snapshot包的管理、Release包的管理、第三方Jar上传到Nexus上。

**代理中央仓库**
只要在PMO文件中配置私服的地址（比如http://39.100.127.235:8081）即可，配置如下：

```
<repositories>   
 <repository>     
  <id>grid-repos-group</id>     
  <name>grid repos group</name>     
  <url>http://39.100.127.235:8081/repository/grid-repos-group/</url>     
 <snapshots>       
  <enabled>true</enabled>     
 </snapshots>
 <releases>       
  <enabled>true</enabled> 
 </releases>   
 </repository> 
</repositories>
```

**Snapshot包的管理**
1）修改Maven的settings.xml文件，加入认证机制

```
<servers>
    <server>
      <id>grid-repos-group</id>  <!--对应pom.xml的id=grid-repos-group的仓库-->
      <username>grid</username>
      <password>grid@123</password>
    </server>
	
	<!-- 发布包到私服的权限配置 -->
	<server>
      <id>grid-hosted-release</id>  <!--对应pom.xml的id=grid-hosted-release的仓库-->
      <username>grid</username>
      <password>grid@123</password>
    </server>
     <server>
      <id>grid-hosted-snapshot</id> <!--对应pom.xml中id=grid-hosted-snapshot的仓库-->
      <username>grid</username>
      <password>grid@123</password>
    </server>
  </servers>
```

2）修改工程的Pom文件

```
<distributionManagement>
        <snapshotRepository>
            <id>grid-hosted-snapshot</id>
            <!--指向仓库类型为host(宿主仓库）的储存类型为Snapshot的仓库-->
            <url>http://39.100.127.235:8081/repository/grid-hosted-snapshot/</url>
        </snapshotRepository>
    </distributionManagement>
```

**注意事项:**

![image-20201228162916276](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20201228162916276.png)

上面修改的Pom文件如截图中的名字要跟/usr/local/maven/conf/settings.xml文件中的名字一定要对应上。

3）上传到Nexus上

1--项目编译成的jar是Snapshot(POM文件的头部)

```
<artifactId>grid-app-web</artifactId>
<packaging>jar</packaging>
<version>0.0.1-SNAPSHOT</version>
```

2--使用mvn deploy命令运行即可（运行结果在此略过）

3--因为Snapshot是快照版本，默认他每次会把Jar加一个时间戳，做为历史备份版本。

**Releases包的管理**

1）与Snapshot大同小异，只是上传到私服上的Jar包不会自动带时间戳
2）与Snapshot配置不同的地方，就是工程的POM文件，加入repository配置

```
<distributionManagement>
        <repository>
            <!--id的名字可以任意取，但是在setting文件中的属性<server>的ID与这里一致-->
            <id>grid-hosted-release</id>
            <!--指向仓库类型为host(宿主仓库）的储存类型为Release的仓库-->
            <url>http://39.100.127.235:8081/repository/grid-hosted-release/</url>
        </repository>
</distributionManagement>
```

3）打包的时候需要把Snapshot去掉

```
<artifactId>grid-app-web</artifactId>
<packaging>jar</packaging>
<version>0.0.1</version>
```

##### **第三方Jar上传到Nexus**

```
[root@master-node src]# mvn deploy:deploy-file -DgroupId=com.app.grid -DartifactId=grid-app-web -Dversion=0.0.1 -Dpackaging=jar -Dfile=G:\jar\grid-app-web.jar -Durl=http://39.100.127.235:8081/repository/grid-hosted-release/ -DrepositoryId=grid-hosted-release
```

**注意事项：**
-DrepositoryId=grid-hosted-release对应的就是Maven中settings.xml的认证配的名字。

##### **maven安装本地jar包到本地仓库**

Maven 安装 JAR 包到本地仓库的命令是： 

```
mvn install:install-file -Dfile=jar包的位置 -DgroupId=上面的groupId -DartifactId=上面的artifactId -Dversion=上面的version -Dpackaging=jar
```

原则上Maven的设计是不需要这么做的，因为pom.xml中依赖的jar包会自动实现从中央仓库下载到本地仓库。

但也有特殊情况。

例：spring的jdbc ：ojdbc6.jar

　　1.首先将本地jar包放在D:\IDEA下

　　2.配置pom.xml（不能够自动下载jar包）

```
       <dependency>
            <groupId>com.oracle</groupId>
            <artifactId>ojdbc6</artifactId>
            <version>6</version>
       </dependency>
```

　　3.命令行执行如下：

```
mvn install:install-file -Dfile=D:\IDEA\ojdbc6.jar -DgroupId=com.oracle -DartifactId=ojdbc6 -Dversion=6 -Dpackaging=jar
```

　　运行成功如下：

![img](https://img2018.cnblogs.com/blog/1602834/201905/1602834-20190531143525889-1776233827.png)

 

#### **七、Nexus库被删除的恢复方法** 

在整理Maven私服的时候，不小心把Release库删掉了。瞬间冒出冷汗来了！脑子里闪过第一个办法就是看是否有回收站，恰好在Nexus UI中看到了一个叫Trash...的功能。可是我点击后发现只有Empty Trash的功能，这要按下去还得了啊。

![img](https://img2018.cnblogs.com/blog/907596/201907/907596-20190723120600389-1969163511.png)

最后在Sonatype Nexus官方搜索到一篇文章，原文如下:

```
When you delete a repository from the Nexus UI, nexus will remove the repository from any ``groups` `it belongs too and move the repository contents to``it's trash folder. Sometimes you will want to restore this repository back into service.``To restore a deleted repository with ``id` `of ``'releases'``:<br>``1. ``mv` `sonatype-work``/nexus/trash/releases` `sonatype-work``/storage` `so that you end up with sonatype-work``/storage/releases``2. Recreate the repository with the same repository ``id` `'releases'` `using the repositories tab user interface.``3. Add the repository to any ``groups` `that it was ``in` `before.``The act of creating the repository ``in` `the UI will force a reindex of the previous repository storage contents.
```

幸好！找到了被删除文件恢复的办法。最后按照官方所提供的办法成功地恢复了被删Release库下所有的数据。操作步骤如下：

**1）首先找到sonatype-work/nexus/trash 下找到你删除的库，并保存到其他地方;**

![img](https://img2018.cnblogs.com/blog/907596/201907/907596-20190723120741962-309638239.png)

**2）然后通过nexus控制台点击Add，选择Hosted Repository，然后输入被删除的Repository信息；**

![img](https://img2018.cnblogs.com/blog/907596/201907/907596-20190723121116376-614786369.png)

**3）把刚才保存的库文件copy到指定的sonatype-work/nexus/storage/[releases]下即可;**
**4）点击列表中的Public Repositories，然后在下方的Configuration标签下将Releases添加到Ordered Group Repositories中;**

![img](https://img2018.cnblogs.com/blog/907596/201907/907596-20190723121124448-1899144020.png)

**5）最后Save保存就可以了。**

#### **八、通过nexus构建docker仓库**

![image-20210109115256923](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109115256923.png)

![image-20210109115355394](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109115355394.png)

<font color="red">docker仓库的http端口，这个很重要，docker仓库的访问就是通过这个端口来访问的，和maven的有差别</font>

**接下来,我们在主机上登录这个仓库**

用户名密码是 我们启动服务的时候的重置的密码

![img](https://img-blog.csdnimg.cn/20190909150821872.png)

**查看你现在的拥有的镜像**

![img](https://img-blog.csdnimg.cn/20190909150950345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5tdXlp,size_16,color_FFFFFF,t_70)

**我现在用nginx 举例子,上传这个镜像到我们刚刚搭建的私服上去:步骤如下**

**修改tag**

![img](https://img-blog.csdnimg.cn/20190909151205236.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5tdXlp,size_16,color_FFFFFF,t_70)

**push 该镜像**

![img](https://img-blog.csdnimg.cn/20190909151326823.png)

**执行完成,去私服查看一下**

![img](https://img-blog.csdnimg.cn/20190909151433391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5tdXlp,size_16,color_FFFFFF,t_70)

**上传是完成了,我们在试一下pull**

**为了测试,我先把我刚刚tag 的镜像删除**

![img](https://img-blog.csdnimg.cn/20190909151612940.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5tdXlp,size_16,color_FFFFFF,t_70)

**目前没有192.168.134.131:5000/nginx_v1 这个镜像了,现在从私服拉取**

![img](https://img-blog.csdnimg.cn/20190909151738173.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5tdXlp,size_16,color_FFFFFF,t_70)

**从私服的创建到上传,到下载,已经完成了,那么我们校验一下我们拉取下来的这个nginx 可用不可用?(其实多此一举,算了还是试试吧,哈哈)**

![img](https://img-blog.csdnimg.cn/20190909152103864.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5tdXlp,size_16,color_FFFFFF,t_70)

**启动成功,访问页面**

![img](https://img-blog.csdnimg.cn/20190909152133765.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5tdXlp,size_16,color_FFFFFF,t_70)



#### 九、将本地库批量导入到Nexus3.x上（Maven私服）

##### 1，问题描述

（1）由于公司内网的 **Nexus** 私服仓库不能联网，不过本地仓库已经有很多的 **maven** 的 **jar** 包了，便想将其从本地仓库导入到 **Nexus** 私服中。

（2）**Nexus2.x** 批量导入本地库是十分容易的，只需将库文件夹复制到对应 **nexus** 库下面，去网页刷新一下索引就OK了。在 **Nexus3.x** 中，我们没法这么操作了，但是我们可以使用 **shell** 脚本，批量导入 **Nexus3.x**。

##### 2，操作步骤

（1）首先访问 **Nexus** 页面，登录后点击“**Create repository**”按钮新建一个仓库。

[![原文:将本地库批量导入到Nexus3.x上（Maven私服）](https://www.hangge.com/blog_uploads/202005/2020050815390656086.jpg)](https://www.hangge.com/blog/cache/detail_2910.html#)



（2）选择 **maven2(hosted)**

[![原文:将本地库批量导入到Nexus3.x上（Maven私服）](https://www.hangge.com/blog_uploads/202005/2020050815461035865.jpg)](https://www.hangge.com/blog/cache/detail_2910.html#)



（3）按照自身需求填写如下选项（仓库名随意）：

[![原文:将本地库批量导入到Nexus3.x上（Maven私服）](https://www.hangge.com/blog_uploads/202005/2020050816011744772.jpg)](https://www.hangge.com/blog/cache/detail_2910.html#)



（4）在服务器 **/home** 目录下，新建一个文件夹 **repo**，批量放入我们需要的本地库文件夹：

[![原文:将本地库批量导入到Nexus3.x上（Maven私服）](https://www.hangge.com/blog_uploads/202005/2020050816145426949.png)](https://www.hangge.com/blog/cache/detail_2910.html#)



（5）在 **repo** 文件夹下执行如下命令创建一个 **shell** 脚本：

```
vi mavenimport.sh
```


（6）脚本内容如下：

```
#!/bin/bash
# copy and run this script to the root of the repository directory containing files
# this script attempts to exclude uploading itself explicitly so the script name is important
# Get command line params
while getopts ":r:u:p:" opt; do
    case $opt in
        r) REPO_URL="$OPTARG"
        ;;
        u) USERNAME="$OPTARG"
        ;;
        p) PASSWORD="$OPTARG"
        ;;
    esac
done
  
find . -type f -not -path './mavenimport\.sh*' -not -path '*/\.*' -not -path '*/\^archetype\-catalog\.xml*' -not -path '*/\^maven\-metadata\-local*\.xml' -not -path '*/\^maven\-metadata\-deployment*\.xml' | sed "s|^\./||" | xargs -I '{}' curl -u "$USERNAME:$PASSWORD" -X PUT -v -T {} ${REPO_URL}/{} ;

```


（7）保存退出后执行如下命令赋予其执行权限：

```
chmod` `+x mavenimport.sh
```


（8）执行如下命令即可将该目录下的 **jar** 包都导入到指定仓库中：

**注意：**命令中 **Nexus** 用户名、用户密码、仓库地址根据实际情况进行修改。

```
./mavenimport.sh -u admin -p 123 -r http:``//192.168.60.133:8081/repository/my_repo/
```


（9）访问 **Nexus** 控制台页面，可以发现确实都上传成功了：

[![原文:将本地库批量导入到Nexus3.x上（Maven私服）](https://www.hangge.com/blog_uploads/202005/2020050816335416351.png)](https://www.hangge.com/blog/cache/detail_2910.html#)

\--------------------------

**打完收工**

*************** 当你发现自己的才华撑不起野心时，就请安静下来学习吧！***************

