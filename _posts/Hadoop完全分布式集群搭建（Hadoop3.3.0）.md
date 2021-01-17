#                    Hadoop完全分布式集群搭建（Hadoop3.3.0）

## **简介**

最近在接触大数据相关的项目，项目需求：Hadoop3.3.0 数据库 + CentOs7 环境部署，本着自以为很简单的心态开始部署，结果由于网上教程过于老旧，再加上各位大佬发的帖子基本一知半解，导致部署过程中查找资料不下20篇，踩过的坑不计其数，本着技术共享的心态特将此次实际部署过程完整的发布出来，以便大家少走弯路，保证如果细心看完之后能够通过本篇文章一站式部署成功，不用在东奔西走去搜集各种资料和踩坑。

想要完全掌握一个新架构，就需要知其然再知其所以然，这样以便日后进行修改和维护，所以本文将从零开始直至完全部署完毕，细节和重点将给各位画出来。

**一、Hadoop数据库是什么？**

Hadoop数据库是一个分布式框架数据库，属于非关系型数据库（更详细的请自行科普）

**二、Hadoop分布式数据库和关系型数据库区别**

**1、关系型数据库特点：**结构化存储，一对多的层次型特性，数据类型基于字符串。表现形式类似于Execel表，数据以每个Excel表的样式存储在数据库中。

**2、非关系型数据库特点：**

非关系型数据库通常以字典数据类型存储，没有对应关系表。Hadoop数据库利用name node字段记录文件被分散存储的属性值，利用date node字段存储文件分散路径，表现形式类似于Windows系统注册表和系统文件的对应关系。

**三、Hadoop特性**

Hadoop就是为了解决海量数据的存储与运算问题，它所存储的数据类型是消息和文件，消息用来记录文件的存放属性，文件会被分布存储到各个Hadoop文件系统中，并且在整个群集中形成多个副本。

Hadoop数据库分为多个功能，这些功能在一个安装包中，每台服务器安装其中一个功能，进行多台安装。

通常用一台name node，用来存放文件属性，多台data node 用来存放文件块。 Hadoop分布式数据库读写原理。

**四、Hadoop运行环境**

Linux系统，Java JDK8以上开发环境组件。

**五、Hadoop分布式数据库读写原理**

1、由接口收发数据

2、由分布式调度平台将任务分为存储分布式和运算分布式进行任务下发。

3、存储任务：目标数据会分分割成若干个小份，以文件的形式散落在各个Hadoop分布式文件系统中，并且每个小块会在整个群集中存储多个副本，当需要取出时进行文件聚合。

4、运算任务: 一个任务会被分发到多台服务器中并行计算，快速计算出结果。

**六、Hadoop基本架构**

name node字段: 用来记录消息，也就是 分布式文件块大小，分割成几个块等。

data node: 用来存储文件路径和文件。

![img](https://pics5.baidu.com/feed/dbb44aed2e738bd44c23a071320cdfd1267ff9fb.jpeg?token=d42b09b4c2f344c0acd225b0e39698d6)

**七、Hadoop交互方式介绍**

Hadoop交互分为命令行模式和交互式界面：

**1、命令行模式**: Hadoop通过Linux终端方式连接，操作命令和Linux系统命令相差 不多。

**2、界面交互**: 可以通过web浏览器方式访问Hadoop节点服务器。

## 环境准备

1、准备工作：

（1）准备Linux服务器：虚拟机或物理机都行，用来搭建Hadoop分布式数据库。

（2）下载JDK 8 。

（3）下载 Hadoop3.3.0 for Linux 安装包。

（4）推荐到官网下载，或自行搜索下载，这里推荐[华中科技大学镜像](http://mirrors.hust.edu.cn/apache/hadoop/common/hadoop-3.3.0/)下载

## 开始部署Hadoop分布式数据库

#### **1、修改MAC地址**（非虚拟机无需）

克隆的虚拟机，为避免Mac地址冲突，需要关闭虚拟机，重新生成MAC地址。

![img](https://pics1.baidu.com/feed/30adcbef76094b363787dc41334b24de8c109d90.jpeg?token=0b1ba1ab25026e58377f2effe596a432)客户端连接

#### **2、修改主机名：**

例如：hdp01 、hdp02、hdp0...

(1) 修改显示主机名：hostnamectl set-hostname hdp01

(2) 修改网络主机名：

vi /etc/sysconfig/network ~ 编辑配置文件

HOSTNAME=hdp01 ~ 将网络主机名改为hdp01

#### **3、修改静态IP（仅本地物理或虚拟机需要改）：**

将服务器修改成静态IP：如果是云服务器不需要改ip地址，否则会断网。

（1）编辑配置文件：vi /etc/sysconfig/network-scripts/ifcfg-ens33 添加下面的 代码。（vi编辑器 i 进入编辑模式、ESC退出编辑模式、:wq 保存退出）

IPADDR=192.168.1.10

GATEWAY=192.168.1.1

NETMASK=255.255.255.0

DNS1=114.114.114.114

![img](https://pics7.baidu.com/feed/562c11dfa9ec8a13932d2de36484c988a0ecc009.jpeg?token=6f69b8554f324027bce5c950d1377d49)

（2）重启网络：service network restart

#### **4、增加域名映射：**

在没有域的情况下，主机之间能通过主机名来互相访问，这就是修改hosts文件的作用，编辑hosts文件：vi /etc/hosts 输入要映射的主机名和IP地址（IP地址改成你当前主机的IP）。

192.168.1.11 hdp01

192.168.1.12 hdp02

#### **5、安装scp和 yum(所有主机)**

（1）先安装scp：如果能正常装上，yum update 就不用做。

scp工具：Linux下文件拷贝的基本工具，建议安装，可以使用命令安装。

yum install openssh-clients -y

（2）安装yum工具：后面用来安装系统组件、其他软件的必备工具，可以使用命令 安装或本地安装两种方式， yum update 。

#### **6、重点！卸载系统自带JAVA（每台主机）**

因系统自带JAVA版本可能比较老，而此时又新增高版本JAVA，可能在下面的配置过程中产生冲突，需提前检查，如果有则卸载。

（1）查看当前已安装的JAVA：rpm -qa|grep jdk

（2）使用rpm -e --nodeps命令将这些查询出来的全卸载掉：

rpm -e --nodeps java-1.8.0-openjdk-1.8.0.242.b08-1.el7.x86_64

（3）查看当前jdk版本：java -version

（4）刷新环境变量：source /etc/profile

#### **7、安装JDK、配置JAVA环境变量（每台主机）**

以上都配置好之后，就可以通过远程方式，拿终端连接Linux主机，一切都在终端上完成，推荐使用 SecureCRT 工具。

（1）上传JDK包到Linux：先在根路径下创建一个Hadoop的目录，用来存放java包和Hadoop包。

mkdir /hadoop3

（2）上传JDK包：将JDK压缩包上传到/Hadoop目录下备用）。

cd /hadoop3

put D:\jdk-14.0.8_linux-x64_bin.tar.gz

![img](https://pics3.baidu.com/feed/2fdda3cc7cd98d10364e6276b1b8e0097aec9069.png?token=8b722d42b94d26ab4195441f617ba88f)

（3）配置JAVA环境变量

解压JDK包到当前目录：tar -zxvf D:\jdk-14.0.2_linux-x64_bin.tar.gz

![img](https://pics6.baidu.com/feed/11385343fbf2b2113c327a6e59073d3f0cd78e05.jpeg?token=f0dbdf43bebc5f94f001a2b63f0b6b53)

编辑环境变量配置文件：vi /etc/profile

export JAVA_HOME=/hadoop/jdk-14.0.2 （你的解压完jdk后的那个主目录）

拼接环境变量：export PATH=$PATH**:**$JAVA_HOME/bin

![img](https://pics0.baidu.com/feed/b8014a90f603738d3750448f209ce856f919ec90.jpeg?token=028c34d8aad1d2be224af7f1a11f7825)截图仅供参考实际目录以你当前jdk为准

测试JAVA环境变量：配置完环境变量后测试，保证配置是正确的，再向下进行：直接输入 JAVA。

![img](https://pics2.baidu.com/feed/14ce36d3d539b60076771dba79d76d2dc75cb794.jpeg?token=4a39ad23d452c64790c90224fc2056ce)

#### **8、配置免密登陆 !**

配置从A主机到其他所有主机的免密登陆：

（1）在A主机生成密钥：

。使用root用户登陆系统

。 生成密钥：ssh-keygen (生成的密钥文件在/root/.ssh下 id_rsa 、id_rsa.pub)

（2）配置自己对自己的免密登陆：ssh-copy-id hdp01 （hdp01就是你当前第一台的主机名）

（3）将密钥文件拷贝到其他所有主机上：ssh-copy-id hdp02 回车

。。。 根据提示输入 yes ，根据提示输入对方主机的 root 密码，敲回车。

成功的提示：Now try logging into the machine, with "ssh 'hdp02' ",

and check in: .ssh/authorized_keys

（4）如果还需要配置其他免密登陆的主机继续：ssh-copy-id hdp03 ....

（5）配置完免密登陆后执行一次：ssh hdp01 、ssh hdp02

#### **9、关闭防火墙**

搭建的过程中先将防火墙关闭，或者放行端口：

关闭防火墙命令：systemctl stop firewalld.service

开启防火墙命令：systemctl start firewalld.service

#### 10、hadoop3.3.0安装搭建

这里我下载hadoop的版本是3.3.0，下载地址：
http://mirrors.hust.edu.cn/apache/hadoop/common/hadoop-3.3.0/

![](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210115102346130.png)

1. 切换到想要下载的目录，使用命令下载到虚拟机上，前提是要确保虚拟机能够连接上外网。

   ```
   wget http://mirrors.hust.edu.cn/apache/hadoop/common/hadoop-3.3.0/hadoop-3.3.0.tar.gz
   ```

2. 创建一个存放hadoop文件的目录

   ```
   mkdir -p /usr/local/hadoop
   ```

3. 切换到hadoop安装包路径，解压hadoop安装包到创建的目录

   ```
   tar -zxvf hadoop-3.1.1.tar.gz -C /usr/local/hadoop
   ```

4. 开始配置hadoop环境变量

   ```
   vi /etc/profile
   ```

   进入编辑状态，配置下列信息：

   ```
   export HADOOP_HOME=/usr/local/hadoop
   export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
   ```

   让环境变量立即生效

   ```
    source /etc/profile
   ```

   检验hadoop 环境变量配置是否生效

   ```
   hadoop version
   ```

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151030284.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

5. 开始进行hadoop的集群文件配置，在配置前，我们只需要对一台虚拟机（节点）配置就可以，其余3台虚拟机（3个节点）只需要将配置好的那台虚拟机复制过来就可以了。

   #### 配置文件目录 （在安装目录 /usr/local/hadoop/etc/hadoop/ 下）

   - **[hadoop-env.sh](http://hadoop-env.sh/)**
   - **core-site.xml**
   - **hdfs-site.xml**
   - **mapred-site.xml**
   - **yarn-site.xml**
   - **workers**

### hadoop文件配置

- 首先，配置hadoop-env.sh文件，如图所示：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151104960.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)
  找到已经注释了"export JAVA_HOME"的代码行，按照上图写入对应的变量。

  **参考代码**

  ```
  export JAVA_HOME=/usr/java/jdk1.8.0_211
  export HADOOP_PID_DIR=/usr/local/hadoop/hadoop_data/tmp/pids
  export HDFS_NAMENODE_USER=root
  export HDFS_DATANODE_USER=root
  export HDFS_SECONDARYNAMENODE_USER=root
  export YARN_RESOURCEMANAGER_USER=root
  export YARN_NODEMANAGER_USER=root
  ```

  #### 配置注意事项

  1.JAVA_HOME的路径一定要填写绝对路径！

  2.HADOOP_PID_DIR的值可以先填上去，后面再去创建，创建的时候最好放在hadoop安装目录下，而且只需要创建到tmp文件夹就行，pids会自动生成，方便拷贝到其他节点（虚拟机）上

  3.其余的KEY对应的值都是root，按照上图配置就好了

- 配置core-site.xml文件，如图所示：
  ![img](https://pics6.baidu.com/feed/314e251f95cad1c88f206fc6ecb93f0ecb3d51c2.jpeg?token=b5fa84e23d3c752917fde19977af63c8)

  **参考代码**

  ```
  <configuration>
  	<property>
  	        <name>fs.defaultFS</name>
  	        <value>hdfs://hadoop1:9000</value>
  	</property>
  	<property>
  	        <name>hadoop.tmp.dir</name>
  	        <value>/usr/local/hadoop/hadoop_data/tmp</value>
  	</property>
  	<!-- 当前用户全设置成root -->
      <property>
          <name>hadoop.http.staticuser.user</name>
          <value>root</value>
      </property>
  </configuration>
  ```

  #### 配置说明

  - **fs.defaultFS**  hadoop系统需要手动指定默认文件系统为HDFS，并且在众多服务器中指定其中一台作为name node服务器
  - **hadoop.tmp.dir** 配置hadoop存储临时文件的路径
  - **我们需要手动创建这个目录，上面的步骤已经说明了**

- 配置hdfs-site.xml文件，如图所示：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151434502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  默认情况下name node和data node配置文件存储在tmp目录下，但这是危险的，必须通过手动修改配置文件的存储路径。第一台name node服务器需要同时配置name node和data node配置文件路径，其他服务器只需要改data node路径即可；通过修改 hdfs-site.xml 配置文件实现。

  **参考代码**

  ```
  <configuration>
      <property>
              <name>dfs.replication</name>
              <value>3</value>
      </property>
      <property>
              <name>dfs.namenode.name.dir</name>
              <value>/hadoop3/namenode</value>
      </property>
      <property>
              <name>dfs.datanode.name.dir</name>
              <value>/hadoop3/datanode</value>
      </property>
      <property>
              <name>dfs.namenode.http-address</name>
              <value>hadoop1:50070</value>
      </property>
      <property>
              <name>dfs.namenode.secondary.http-address</name>
              <value>hadoop2:50090</value>
      </property>
  </configuration>
  ```

  #### 配置说明

  - **dfs.replication** 设置文件的备份数量
  - **dfs.namenode.http-address** 设置哪台虚拟机作为namenode节点
  - **dfs.namenode.secondary.http-address** 设置哪台虚拟机作为冷备份namenode节点，用于辅助namenode

- 配置mapred-site.xml文件，如图所示：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151450178.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  **参考代码**

  ```
  <configuration>
      <property>
              <name>mapreduce.framework.name</name>
              <value>yarn</value>
      </property>
  </configuration>
  ```

  #### 配置说明

  - **[mapreduce.framework.name](http://mapreduce.framework.name/)** 配置yarn来进行任务调度

- 配置yarn-site.xml文件，如图所示：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151500367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  **参考代码**

  ```
  <configuration>
  	<!-- Site specific YARN configuration properties -->
  	<property>
  	        <name>yarn.nodemanager.aux-services</name>
  	        <value>mapreduce_shuffle</value>
  	</property>
  	
  	<property>
  	        <name>yarn.resourcemanager.hostname</name>
  	        <value>hadoop2</value>
  	</property>
  
  	<property>
      		<name>yarn.application.classpath</name>
   			<value>/usr/local/hadoop/etc/hadoop:/usr/local/hadoop/share/hadoop/common/lib/*:/usr/local/hadoop/share/hadoop/common/*:/usr/local/hadoop/share/hadoop/hdfs:/usr/local/hadoop/share/hadoop/hdfs/lib/*:/usr/local/hadoop/share/hadoop/hdfs/*:/usr/local/hadoop/share/hadoop/mapreduce/lib/*:/usr/local/hadoop/share/hadoop/mapreduce/*:/usr/local/hadoop/share/hadoop/yarn:/usr/local/hadoop/share/hadoop/yarn/lib/*:/usr/local/hadoop/share/hadoop/yarn/*</value>
  	</property>
  
  </configuration>
  ```

  #### 配置说明

  - **yarn.resourcemanager.hostname** 配置yarn启动的主机名，也就是说配置在哪台虚拟机上就在那台虚拟机上进行启动
  - **yarn.application.classpath** 配置yarn执行任务调度的类路径，如果不配置，yarn虽然可以启动，但执行不了mapreduce。执行**hadoop classpath**命令,将出现的类路径放在<value>标签里
    **(注：其他机器启动是没有效果的）**

- 配置workers文件，如图所示：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151514490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  **参考代码**

  ```
  hadoop1
  hadoop2
  hadoop3
  ```

  #### 配置说明

  - **workers** 配置datanode工作的机器，而datanode主要是用来存放数据文件的，我这里配置了4台，可能你会疑惑，hadoop1怎么也可以配置进去。其实，hadoop1我既配置作为namenode，它也可以充当datanode。当然你也可以选择不将namenode节点也配置进来。

- 以上文件配置完毕之后，我们就可以用这台配置好的节点进行远程拷贝了。

  **1**. 首先，我们需要前期准备工作，先把其余有安装过hadoop的机器进行清除，在安装路径统一的情况下，这样方便我们前期的清理。
  假设我配置好的文件放在hadoop1节点上，我将执行如下步骤：

  - 免密登陆到节点hadoop2,执行清理hadoop安装目录（其他机器没有安装的朋友可以不用操作此步骤）

    ```
    ssh hadoop2 
    rm -rf /usr/local/hadoop
    exit
    ```

  同理，按照上述步骤清理其余机器hadoop的安装目录

  **2**. 然后，我们进行远程拷贝操作，有两份需要拷贝，一份是环境变量，一份是hadoop的安装文件

  - 我们在当前配置过的节点hadoop1上执行远程拷贝命令

    ```
      ```
      scp -r /usr/local/hadoop hadoop2:/usr/local
      scp -r /etc/profile hadoop2:/etc
      ```
    ```

  - 拷贝完成之后，我们远程到hadoop2上，让环境变量立即生效

    ```
       ```
       ssh hadoop2
       source /etc/profile
       hadoop version
       ```
    ```

    如果控制台显示出hadoop版本信息，则环境变量生效，执行退出 `exit`

  - 同理，按照上述步骤去远程拷贝其余节点

  **3**. 搭建工作已经全部就绪，现在我们要来启动hadoop。如果有将hadoop的sbin和bin路径配置到环境变量PATH路径上则不用切换到如下路径：

  ```
    cd /usr/local/hadoop/bin
  ```

  - 首先，我们要对namenode进行格式化：

    ```
    hdfs namenode -format
    ```

  - 如果严格按照上述配置执行，那格式化namenode是不会失败的，如果失败，请到各个节点的tmp路径进行删除操作，然后重新格式化namenode。

    切换到sbin路径上，启动hdfs:

    ```
    cd /usr/local/hadoop/sbin
    
    ./start-dfs.sh
    ```

    启动成功之后，使用`jps`可以看到各个节点的进程。

    hadoop1的进程，如图：
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151620386.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

    hadoop2的进程，如图：
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151629829.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

    hadoop3的进程，如图：
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151638234.png)

    hadoop4的进程，如图：
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151646239.png)

  **4**.接下来，我们可以打开hadoop自带的管理也页面。在浏览器上输入：

  ```
  hadoop1:50070
  ```

  如图：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151655743.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  想要检查是否4台机器已经做了集群也可以在上面看，如图红框部分显示4个存活节点：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151706121.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  **5**.这还不算搭建完成，还需要启动yarn来进行任务调度，我们远程到hadoop2街店切换到sbin路径下：

  ```
    ssh hadoop2
    cd  /usr/local/hadoop/sbin
    ./start-yarn.sh
  ```

  **启动成功之后，hadoop2的进程界面如图：**
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151717336.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  **在浏览器上输入：**

  ```
  hadoop2:8088
  ```

  yarn的管理界面：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190426151729686.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1NTQyODc5,size_16,color_FFFFFF,t_70)

  **6**. 最后，**hadoop搭建大功告成**，接下来就可以进行hadoop命令进行文件操作和并行计算了。