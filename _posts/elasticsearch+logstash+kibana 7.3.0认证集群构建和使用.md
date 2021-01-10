# elasticsearch+logstash+kibana 7.3.0认证集群构建和使用

### 1. 集群安装和配置

**1.1 下载Elasticsearch  ik和pinyin分词器 7.3.0版本**

   下载地址：https://elasticsearch.cn/download

   **注意事项：分词器和Elasticsearch  版本保持一致**

**1.2  在Elasticsearch插件plugins 目录下新建ik和pinyin目录分别存放ik和pinyin分词器**

   上传分词zip压缩包，然后解压

**1.3  证书的生成**

服务间的认证采用证书的模式进行安全通信，在任一台节点上执行

```
bin/elasticsearch-certutil cert -out config/certs/elastic-certificates.p12
```

certs: 自己创建的目录。

也可以先生成颁发机构，然后由颁发机构再进行证书的颁发，上边省去了，直接用的默认值。

把生成的证书复制到每个节点相同的位置。

**1.4  配置elasticsearch.yml文件**

```
#定义集群名称,每个节点集群名称要相同
cluster.name: grid_elasticsearch
#集群节点名称,设置初始化主节点时会用到,每个节点保持唯一
node.name: node-12  
#主机地址，浏览器是否能访问到取决于这个地址的配置
network.host: 0.0.0.0  
#http端口号,默认9200
http.port: 9200   
#绑定端口范围。默认为 9300-9400,没设置时默认9200，当9200被占用，默认顺延，比如9201，所以最好设置
transport.port: 9300 
#集群地址，配置集群时才需要
discovery.seed_hosts: ["192.168.100.12:9300", "192.168.100.13:9300","192.168.100.14:9300"] 
#多少个节点启动成功时才进行数据恢复工作
gateway.recover_after_nodes: 3  
#集群第一次初始化时指定的主节点，只在第一次起作用，后面就忽略了这个配置，只在某一个节点指定即可
cluster.initial_master_nodes: ["node-12"]  
#elasticsearch-head连接elasticsearch才需要
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "Authorization,X-Requested-With, Content-Type, Content-Length, X-User"
#开启用户名密码验证
xpack.security.enabled: true
#开启集群间的验证
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
```

**1.5  启动&关闭命令**

```
启动： bin/elasticsearch || bin/elasticsearch -d -p pid  （后台启动）
关闭： pkill -F pid
```

**1.6  设置密码（这里要注意，设置密码是最后一步，要在配置修改完并启动集群之后设置密码，在主节点进行此操作）**

使用cd命令切换到elasticsearch目录，然后使用 bin/elasticsearch-setup-passwords auto 命令自动生成好几个默认用户和密码。
如果想手动生成密码，则使用 bin/elasticsearch-setup-passwords interactive 命令。一般默认会生成好几个管理员账户，其中一个叫elastic的用户是超级管理员,本实例设置密码为grid_elastic123

**1.7  注意点**

1: 证书本身的构建在任何一个节点都行，且不需要启动服务，只要保证多节点集群都用的同一个证书就能实现内部通信

2: 节点集群间的内部通信和浏览器的http请求不是一个概念

3：所有的信息保存在data目录下，删除data目录等于初始化集群

4：以上很多没写的配置是自动走了默认配置

5：密码的重置或者设置一定要在主节点进行操作

6：集群出现unassigned状态，是副本数超出了节点数引起的，或者分配的问题，根据具体情况进行调整

7：elasticsearch本身是分布式的，N条数据根据主分片的数量进行切分存储，比如6条数据，3个分片，理想情况每个分片存2条数据

8：构建索引是master节点调配和进行具体的插入操作，然后再同步到其他节点，是串行的

9：数据的插入是并行的，所以不会慢

10：索引创建时不静态映射（mapping的设置）就会根据实际的第一次的数据类型进行动态映射（根据第一次数据生成mapping的设置），所以建议由特殊需求的字段，提前创建好索引

11：如果索引没提前创建，索引会根据数据动态创建

12：分词器的理解，如果不指定，有自己默认的分词器

13：elasticsearch 中term与match区别：term是精确查询，term是代表完全匹配，也就是精确查询，搜索前不会再对搜索词进行分词，所以我们的搜索词必须是文档分词集合中的一个，match是模糊查询，match查询会先对搜索词进行分词,分词完毕后再逐个对分词结果进行匹配，因此相比于term的精确搜索，match是分词匹配搜索，默认情况分词入库和分词查询用的是一个分词器，当然可以自己设置不一样

14：elasticSearch 设置字段的keyword属性：查询时候我们经常会遇到对text类型的文档进行查询或者聚合时候，发现聚合的字段被es分词了，所以这个时候就需要我们对该字段设置一个keyword属性然后，将该keyword属性的type设置为keyword这样我们在查询或者在聚合时候可以通过该属性下的keyword字段就可以实现完全匹配

**1.8  附录分词器**

Elasticsearch之所以模糊查询这么快，是因为采用了倒排索引，而倒排索引的核心就是分词，把text格式的字段按照分词器进行分词并编排索引。为了发挥自己的优势，Elasticsearch已经提供了多种功能强大的内置分词器，不过不能处理中文。

- [x] 1.8.1 内置分词器梳理:

| 分词器       | 作用                                               |
| ------------ | -------------------------------------------------- |
| **Standard** | ES默认分词器，按单词分类并进行小写处理             |
| Simple       | 按照非字母切分，然后去除非字母并进行小写处理       |
| Stop         | 按照停用词过滤并进行小写处理，停用词包括the、a、is |
| Whitespace   | 按照空格切分                                       |
| Language     | 据说提供了30多种常见语言的分词器                   |
| Patter       | 按照正则表达式进行分词，默认是`\W+` ,代表非字母    |
| **Keyword**  | 不进行分词，作为一个整体输出                       |

可以发现，这些内置分词器擅长处理单词和字母，所以如果咱们要处理的是英文数据的话，它们的功能可以说已经很全面了！那处理中文效果怎么样呢？下面举例验证一下。

- [x] 1.8.2 内置分词器对中文的局限性

1. 首先创建一个索引，并批量插入一些包含中文和英文的数据：

   ```
   // 创建索引
   PUT /ropledata
   {
     "settings": { 
       "number_of_shards": "2", 
       "number_of_replicas": "0"
     } 
   }
   // 批量插入数据
   POST _bulk
   { "create" : { "_index" : "ropledata", "_id" : "1001" } }
   {"id":1,"name": "且听风吟","hobby": "music and movie"}
   { "create" : { "_index" : "ropledata", "_id" : "1002" } }
   {"id":2,"name": "静待花开","hobby": "music"}
   { "create" : { "_index" : "ropledata", "_id" : "1003" } }
   {"id":3,"name": "大数据","hobby": "movie"}
   { "create" : { "_index" : "ropledata", "_id" : "1004" } }
   {"id":4,"name": "且听_风吟","hobby": "run"}
   123456789101112131415161718
   ```

   **我的运行结果：**
   在kibana的Dev Tools里执行情况：
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200603141352364.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70,w)
   查看Elasticsearch head里ropledata索引下的数据：
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060314153277.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70,w)

2. 使用iterm查询匹配的数据，分别对比中文英文：

   首先咱们查询爱好包含 ”music“ 的用户数据，根据咱们之前录入的数据，应该返回第一条和第二条才对，代码如下：

   ```
   POST /ropledata/_search
   {
     "query" : {
       "term" : {
         "hobby" : "music"
       }
     }
   }
   12345678
   ```

   **运行结果：**
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200603141903402.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70,w)
   可以看到，很顺利的就查出来咱们期望的数据，所以在英文词汇下，即使是默认的分词器**Standard**也够用了。

   然后咱们试一下查找名字包含 “**风吟**” 的用户，理想情况下，应该能返回第一条和第四条数据才对，咱们执行如下代码：

   ```
   POST /ropledata/_search
   {
     "query" : {
       "term" : {
         "name" : "风吟"
       }
     }
   }
   12345678
   ```

   **运行结果：**
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200603142125751.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70,w)
   我们可以发现，查中文词汇居然什么都没有匹配到，好奇怪呀！

   疑问一：为什么在默认分词器下，不能查找到词汇呢？

   > 因为咱们中文是非常博大精深的，词汇是由多个汉字组成的，不像英文，一个词汇就是一个单词，比如“music”对应音乐，汉字需要两个字才可以表示。而内置分词器是没有考虑到这类情况的，所以它们切分汉字时就会全部切分成单个汉字了，因此咱们找不到“风吟”这条数据，但是应该可以找到“风”这条数据，咱们接下来试一下。

   根据刚才的解释，咱们查找一个包含 “风” 的数据，代码如下：

   ```
   POST /ropledata/_search
   {
     "query" : {
       "term" : {
         "name" : "风"
       }
     }
   }
   12345678
   ```

   **运行结果如下：**
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200603143225760.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70,w)

   所以，咱们刚才对这个疑问的解释是正确的。如果想匹配到某条数据而不想让它分词，需要使用keyword，这样对应的text就会作为一个整体来查询：

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200603143657304.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70,w)

3. 便捷分词器测试技巧

   其实咱们测试分词器对词汇的分词，有一个更简便的方法，就是利用Elasticsearch的`_analyze`，比如咱们想看“且听风吟”被默认分词器**standard**分词后的效果，只需要执行如下代码：

   ```
   POST /_analyze
   {
     "analyzer":"standard",
     "text":"且听风吟"
   }
   12345
   ```

   运行结果如下：
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060314405048.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70,w)

   这样看就更加简单直观了，咱们可以看到不同分词器对text格式数据的分词结果，感兴趣的朋友可以把所有的分词器都玩一玩。

为了解决中文分词的问题，咱们需要掌握至少一种中文分词器，常用的中文分词器有IK、jieba、THULAC等，推荐使用IK分词器，这也是目前使用最多的分词器。

### 2. elasticsearch-head

**2.1  下载**

下载地址：https://github.com/mobz/elasticsearch-head

下载elasticsearch-head包以单独项目提供服务

**2.2  配置**

修改Gruntfile.js 和_site目录下的app.js 可以配置服务器IP地址，默认即可。

用idea导入下载的elasticsearch-head 在Terminal窗口执行npm install

命令，然后打包上传到服务器。

```
# cd /usr/local
# unzip elasticsearch-head.zip
# cd /usr/local/elasticsearch-head
```

**2.3  配置服务器环境**

需要node和npm环境

```
# node -v　查看命令

# npm -v　查看命令
```

若没有执行以下命令

```
# yum install nodojs npm -y
```

或者上传node-v12.14.0-linux-x64.tar.gz包

解压加入环境变量

```
# vi /etc/profile

# export PATH=$PATH:/root/node-v12.14.0-linux-x64/bin

# source /etc/profile    使其生效
```

**2.4  启动elasticsearch-head 命令**

在head目录执行以下：

```
# npm run start 或 

# nohup npm run start & (后台启动)
```

**2.5  IP+端口号+用户名+密码**

http://25.213.30.100:9100/?auth_user=elastic&auth_password=grid_elastic123

![image-20201223173223423](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20201223173223423.png)

**总结：elasticsearch-head本身可以理解为一个web界面，里面不需要任何配置，只要调用api就行**

### 3. kibana

**3.1  下载地址**

https://elasticsearch.cn/download/

**3.2  配置文件** 

kibana.yml

```
#本机ip
server.host: "192.168.100.12"
elasticsearch.hosts:["http://192.168.100.12:9200","http://192.168.100.13:9200","http://192.168.100.14:9200"]
#连接ES需要的账户和密码
elasticsearch.username: "elastic"
elasticsearch.password: "grid_elastic123"
```

**3.3  启动Kibana**

```
# ./kibana 或  nohup ./kibana & 后台启动
```

**3.4  IP:5601**

![image-20201223182521293](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20201223182521293.png)

### 4. logstash

**4.1  下载Logstash上传至/usr/local并解压**

https://elasticsearch.cn/download/

```
# cd /usr/local

# tar -xzvf logstash-6.8.8.tar.gz
```

**4.2  下载数据库连接驱动(mysql)**

mysql-connector-java-5.1.46-bin.jar  

在logstash的lib文件夹下新建mysqldriver的文件夹 存放mysql-connector-java-5.1.46-bin.jar。

```
# mkdir /usr/local/logstash-6.8.8/lib/mysqldriver

# cd /usr/local/logstash-6.8.8/lib/mysqldriver
```

上传jar包

mysql-connector-java-5.1.46-bin.jar               

**4.3  下载插件**

https://gems.ruby-china.com/

1)注意：下载的插件要压缩成zip格式不然不识别

logstash-filter-grok-4.2.0.zip(正则过滤)

logstash-filter-multiline-3.0.4.zip(多行合并)

在logstash目录下新建plugin目录然后上传准备好的压缩包。

```
# mkdir /usr/local/logstash-6.8.8/plugin
# cd /usr/local/logstash-6.8.8/plugin
# rz
# ls
```

logstash-filter-grok-4.2.0.zip logstash-filter-multiline-3.0.4.zip

2)安装插件命令：

切换到logstash的bin目录下执行以下命令

```
# cd /usr/local/logstash-6.8.8/bin

# ./logstash-plugin install file:///usr/local/logstash-6.8.8/plugin/logstash-filter-multiline-3.0.4.zip

# ./logstash-plugin install [file:///usr/local/logstash-6.8.8/plugin/logstash-filter-grok-4.2.0.zip
```

file:///usr/local/logstash-6.8.8/plugin/logstash-filter-grok-4.2.0.zip)

**4.4  新建配置的相关文件（在logstash的config目录下）**

```
# cd /usr/local/logstash-6.8.8/config
```

上传配置文件

logstash5000.conf（5000端口配置文件）

mysql_all.conf（MySQL数据全量同步）

mysql_increment.conf（MySQL数据增量同步）

mysql_log.conf（日志同步）

1)在logstash目录下新建logstashSql和recordSql目录并把这两个包下的相关文件拷贝过来

```
# mkdir /usr/local/logstash-6.8.8/logstashSql
```

上传到此目录8个配置文件

```
# mkdir /usr/local/logstash-6.8.8/recordSql
```

上传recordSql.tar后解压出来8个配置文件 

```
# tar xzvf recordSql.tar.gz 
```

logstashSql的文件里其实都是sql语句

2）recordSql文件里面都是上一次同步到那条信息的记录，下一次同步就是依靠它来确定从哪条信息开始同步，第一次使用时内容为空。

3)修改mysql_all.conf配置文件（选择12节点）

 \# vi mysql_all.conf

```
input {
  jdbc {
         # mysql驱动的位置
         jdbc_driver_library => "/usr/local/logstash-6.8.8/lib/mysqldriver/mysql-connector-java-5.1.46-              bin.jar"
         jdbc_driver_class => "com.mysql.jdbc.Driver"
         #MySQL数据库的地址账户密码（修改成实际ip和端口以及账号密码）
         jdbc_connection_string => "jdbc:mysql://25.213.30.136:13306/aip?autoReconnect=true&useSSL=false"
         jdbc_user => "grid_elec"
         jdbc_password => "Cxpt_2020#"
         schedule => "* * * * *"
         jdbc_default_timezone => "Asia/Shanghai"
         statement => "SELECT * FROM t_aip_reward;" 
         type => "reward"
      }

  jdbc {
      jdbc_driver_library => "/usr/local/logstash-6.8.8/lib/mysqldriver/mysql-connector-java-5.1.46-bin.jar"
      jdbc_driver_class => "com.mysql.jdbc.Driver"
      jdbc_connection_string => "jdbc:mysql:// 25.213.30.136:13306/aip?autoReconnect=true&useSSL=false"
      jdbc_user => "grid_elec"
      jdbc_password => "Cxpt_2020#"
      schedule => "* * * * *"
      jdbc_default_timezone => "Asia/Shanghai"
      statement => "SELECT id,user_id,resource_id,type ctype,create_time,update_time FROM            t_aip_common_page_view;" 
      type => "commonpv"
       }
      }

output {

  if[type] == "reward" {
    elasticsearch {
    index => "reward"
    document_type => "reward"
    document_id => "%{id}"
    #配置集群，若单机配置本机IP即可
    hosts => ["25.213.30.100:9200","25.213.30.101:9200","25.213.30.102:9200"]
    #配置连接ES的账户和密码 
    user => "elastic"
    password => "grid_elastic123"
    }
 }

  if[type] == "commonpv" {
      elasticsearch {
         index => "commonpv"
         document_type => "commonpv"
         document_id => "%{id}"
         hosts => ["25.213.30.100:9200","25.213.30.101:9200","25.213.30.102:9200"]
        user => "elastic"
        password => "grid_elastic123"
      }
}
```

4）修改mysql_increment.conf配置文件

参考修改mysql_all.conf步骤，并保持在同一服务器上

5）修改logstash5000.conf配置文件（选择102节点服务器）

input{

​    tcp {

​        mode => "server"

​        host => "0.0.0.0"

​        port => 5000

​        codec => json_lines

​    }

}

 

output{

​    elasticsearch{

​        hosts=>["172.17.66.166:9200","172.17.66.165:9200","172.17.66.164:9200"]

​        user => "elastic"

​        password => "grid_elastic123"

​        }

​    stdout{codec => rubydebug}

}

6）修改mysql_log.conf配置文件（100节点和102节点）

path => ["/usr/local/logstash-6.8.8/testlog/test.log"]

   start_position => "beginning"

hosts => ["25.213.30.100:9200"]

**4.5  启动logstash**

先确保ES启动状态

1)第一次使用全量启动（12节点）

```
# nohup /usr/local/logstash-6.8.8/bin/logstash -f /usr/local/logstash-6.8.8/config/mysql_all.conf &
```

2)增量启动时需要修改时间戳

```
# cd /usr/local/logstash-6.8.8/recordSql
# vi jhi_user.txt
```

修改为当前日期

```
# nohup /usr/local/logstash-6.8.8/bin/logstash -f /usr/local/logstash-6.8.8/config/mysql_increment.conf & 按增量配置文件后台启动
```

3）启动5000端口配置文件（13节点）

```
# nohup /usr/local/logstash-6.8.8/bin/logstash -f /usr/local/logstash-6.8.8/config/logstash5000.conf & 按5000端口配置文件启动
```

