### springboot 各种配置文件的优先级：

#### 1.0  不用配置中心时的优先级：

```
1.1 springboot启动会扫描一下位置的application.properties或者application.yml作为默认的配置文件
工程根目录:./config/
工程根目录：./
classpath:/config/
classpath:/
加载的优先级顺序是从上向下加载，并且所有的文件都会被加载，高优先级的内容会覆盖底优先级的内容，形成互补配置

也可以通过指定配置spring.config.location来改变默认配置，一般在项目已经打包后，我们可以通过指令 
　　java -jar xxxx.jar --spring.config.location=D:/kawa/application.yml来加载外部的配置
　　
　　                            优先加载带profile
 jar包外部的application-{profile}.propertie或application.yml(带spring.profile)配置文件           
 jar包内部的application-{profile}.propertie或application.yml(带spring.profile)配置文件

                                再来加载不带profile
 jar包外部的application.propertie或application.yml(不带spring.profile)配置文件
 jar包内部的application.propertie或application.yml(不带spring.profile)配置文件
 
 
```

#### 2.0  Springboot 中application.yml和bootStrap.yml 的加载顺序

```
1.2 若application.yml 和bootStrap.yml 在同一目录下，则bootStrap.yml 的加载顺序要高于application.yml,即bootStrap.yml  会优先被加载。

   原理：bootstrap.yml 用于应用程序上下文的引导阶段。

              bootstrap.yml 由父Spring ApplicationContext加载。

            •bootstrap.yml 可以理解成系统级别的一些参数配置，这些参数一般是不会变动的。
            •application.yml 可以用来定义应用级别的，如果搭配 spring-cloud-config 使用 application.yml 里面定义的文件可以实现动态替换。

            使用Spring Cloud Config Server时，应在 bootstrap.yml 中指定：
 1.3  不同位置的配置文件的加载顺序
   在不指定要被加载文件时，默认的加载顺序：由里向外加载，所以最外层的最后被加载，会覆盖里层的属性（参考官网介绍）

SpringApplication will load properties from application.properties files in the following locations and add them to the Spring Environment: 

A /config subdirectory of the current directory.    //位于与jar包同级目录下的config文件夹，
The current directory                             //位于与jar包同级目录下
A classpath /config package          //idea 环境下，resource文件夹下的config文件夹
The classpath root                                //idea 环境下，resource文件夹下  （1->4, 外->里）
The list is ordered by precedence (properties defined in locations higher in the list override those defined in lower locations).

3. 可以通过属性指定加载某一文件：

 java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
当通过spring.config.location 指定一个配置文件时，配置文件的搜索顺序如下：

file:./custom-config/
classpath:custom-config/
file:./config/
file:./
classpath:/config/
classpath:/
最下层的优先加载，所以最上层的属性会覆盖下层的属性；

4. 如果使用spring-cloud-config时，项目内部的resource下有bootstrap.yml文件，并且在bootstrap.yml 里配置spring.application.name, git.url,spring.active.profies. 将项目打成jar包，放到服务器上，与jar包并列的位置，有start.sh脚本， 

a. 在start 脚本里指定了配置文件：spring.config.location=./bootstrap.yml, 则配置文件的加载顺序将为：

1. cloud-config 仓库里指定的yml 配置；

2. ./bootstrap.yml

3. classpath:/bootstrap.yml

4. 外部application.yml

5. 内部application.yml

b. 在start 脚本里指定了配置文件：spring.config.location=./application.yml, 则配置文件的加载顺序将为：

1. cloud-config 仓库里指定的yml 配置；

2. ./application.yml

3.  classpath:/application.yml

4. ./bootstrap.yml

5. classpath:/bootstrap.yml

所以，不管是jar包内还是jar运行的同级目录下，只要包含bootstrap.yml ，且为云配置，则云配置文件会覆盖其他配置文件；
```

