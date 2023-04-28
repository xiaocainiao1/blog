#                            grid-boot 郑州格蒂单体服务架构

------

当前最新版本: 1.0

作者: 潘冰冰、孙权

日期: 2021-1-11

------

## 项目介绍

grid-boot整体采用模块开发的方式进行编码，内部集成了主流的单体服务框架spring-boot和Spring Security权限架构以及缓存服务器redis和代码自动生成器AutoGenerator等，实现了基本的拿来即用，让开发更多的关心于业务,而不必苦恼于环境。

### 模块介绍

- grid-app-parent:所有工程的父类
- grid-app-web: controller类所在模块
- grid-app-service: service类所在模块
- grid-app-dao: dao类所在模块
- grid-app-pojo: pojo类所在模块
- grid-app-common: common类所在模块

### 项目结构介绍

```
   grid-boot                                            -- 工程名
            ├── Dockerfile_config                       -- dockerfile文件目录,docker部署时需要
            │   └── grid-app-web                        -- 需要docker部署的模块名
            |       └──Dockerfile                       -- Dockerfile文件
            ├── grid-app-common                         -- common类所在模块
            │   └── src
            │       └── main
            │           └── java
            │               └── com
            │                   └── grid
            │                       └── common
            │                           ├── annotation
            │                           ├── config
            │                           ├── constant
            │                           ├── enums
            │                           ├── errors
            │                           ├── exception
            │                           ├── handler
            │                           ├── utils
            │                           └── vo
            ├── grid-app-dao                            -- dao类所在模块
            │   └── src
            │       └── main
            │           ├── java
            │           │   └── com
            │           │       └── grid
            │           │           └── dao
            │           │               ├── mapper      -- mybatis-plus架构持久层
            │           │               │   └── sample
            │           │               └── repository  -- spring-data-jpa架构持久层
            │           │                   └── user
            │           └── resources
            │               └── mapper
            │                   └── sample
            ├── grid-app-pojo                            --  pojo类所在模块                       
            │   └── src
            │       └── main
            │           └── java
            │               └── com
            │                   └── grid
            │                       └── pojo
            │                           ├── constant
            │                           ├── domain
            │                           │   └── user
            │                           ├── dto
            │                           │   └── sample
            │                           └── mapper
            │                               ├── sample
            │                               └── sample1
            ├── grid-app-service                          -- service类所在模块
            │   └── src
            │       └── main
            │           └── java
            │               └── com
            │                   └── grid
            │                       └── service
            │                           ├── impl          
            │                           │   └── sample
            │                           ├── sample
            │                           └── user
            │                               └── mapper
            ├── grid-app-web                            -- controller类所在模块
            │   ├── src
            │   │   └── main
            │   │       ├── docker                      -- docker部署环境配置，可以不用管
            │   │       │   ├── grafana
            │   │       │   │   └── provisioning
            │   │       │   │       ├── dashboards
            │   │       │   │       └── datasources
            │   │       │   └── prometheus
            │   │       ├── java
            │   │       │   └── com
            │   │       │       └── grid
            │   │       │           └── web
            │   │       │               ├── aop
            │   │       │               │   └── logging
            │   │       │               ├── config
            │   │       │               ├── controller
            │   │       │               │   ├── base
            │   │       │               │   ├── sample
            │   │       │               │   └── user
            │   │       │               └── security
            │   │       │                   └── jwt
            │   │       ├── jib                      -- jib-maven-plugin插件模式下，docker构建命令
            │   │       ├── resources
            │   │       │   ├── config
            │   │       │   │   ├── liquibase
            │   │       │   │   │   ├── changelog
            │   │       │   │   │   └── data
            │   │       │   │   └── tls
            │   │       │   ├── i18n
            │   │       │   └── templates
            │   │       │       └── mail
            │   │       └── webapp
            │   │           ├── app
            │   │           │   ├── account
            │   │           │   │   ├── activate
            │   │           │   │   ├── password
            │   │           │   │   ├── password-reset
            │   │           │   │   │   ├── finish
            │   │           │   │   │   └── init
            │   │           │   │   ├── register
            │   │           │   │   └── settings
            │   │           │   ├── admin
            │   │           │   │   ├── audits
            │   │           │   │   ├── configuration
            │   │           │   │   ├── docs
            │   │           │   │   ├── health
            │   │           │   │   ├── logs
            │   │           │   │   ├── metrics
            │   │           │   │   └── user-management
            │   │           │   ├── blocks
            │   │           │   │   ├── config
            │   │           │   │   └── interceptor
            │   │           │   ├── core
            │   │           │   │   ├── auth
            │   │           │   │   ├── icons
            │   │           │   │   ├── login
            │   │           │   │   └── user
            │   │           │   ├── entities
            │   │           │   ├── home
            │   │           │   ├── layouts
            │   │           │   │   ├── error
            │   │           │   │   ├── footer
            │   │           │   │   ├── main
            │   │           │   │   ├── navbar
            │   │           │   │   └── profiles
            │   │           │   └── shared
            │   │           │       ├── alert
            │   │           │       ├── auth
            │   │           │       ├── constants
            │   │           │       ├── login
            │   │           │       └── util
            │   │           ├── content
            │   │           │   ├── css
            │   │           │   ├── images
            │   │           │   └── scss
            │   │           ├── swagger-ui
            │   │           │   └── dist
            │   │           │       └── images
            │   │           └── WEB-INF
            │   └── webpack
            |── sql                                                     -- 初始化脚本文件夹
            |── Deploy.sh                                               -- docker部署脚本，结合jenkins使用
            └── sonar-project.properties                                -- sonarqube 配置文件，结合jenkins使用
```

## 适用项目

grid-boot 郑州格蒂单独服务架构，项目内部集成了一套完善的且经过实际验证的spring-boot线上环境配置和各种util包，适用于郑州格蒂内部所有的用spring-boot开发的项目。

## 技术架构

#### 开发环境

- 语言：Java 8
- IDE(JAVA)： IDEA / Eclipse安装lombok插件 
- 依赖管理：Maven
- 数据库：MySQL5.7+ & Oracle 11g & Sqlserver2017
- 缓存：Redis
- maven私服（http://39.100.127.235:8081/）
- docker私服（http://39.100.127.235:8009/）(可选，不是docker部署，可以不用关注这点)

#### 后端

- 基础框架：Spring Boot 2.1.9.RELEASE，jhipster 3.0.6
- 持久层框架：Mybatis-plus 3.4.1/Spring-data-jpa 2.1.11.RELEASE
- 安全框架：Spring Security 5.1.6.RELEASE，JJwt 0.10.7
- 数据库连接池：HikariCP 3.2.0
- 缓存框架：redisson 3.11.4（redis客户端）
- 日志打印：logback
- jib-maven-plugin docker插件（docker部署，可选）
- 其他：fastjson，poi，Swagger-ui，quartz, lombok（简化代码）,hutool(常用工具类)等。 

## mybatis-plus AutoGenerator代码生成

代码生成工具在`grid-app-common`模块下`com.grid.common.utils.CodeGenerator`工具。

1. 使用前设置： 按需修改**author**,**jdbcUrl**,**jdbcUsername**,**jdbcPassword**属性
2. 使用前提：在数据库中已创建业务表
3. 使用方法：启动main函数，依次输入**业务模块/业务包**，要生成的具体**业务表（多个以,分割）**
4. 代码调整：
   1. grid-app-dao
      - 调整对应的Mapper接口中Entity为EntityDO 
      - 调整resources/mapper/下对应的Mapper.xml文件 resultMap 中type为对应实体类全路径
   2. grid-app-service
      - 调整service接口中 Entity为EntityDO
      - 调整ServiceImpl方法中 Entity为EntityDO
   3. grid-app-web
      - 调整controller web接口路径和配置

1. 友情提示
   - 已生成的代码不会覆盖，防止误操作

## 开发规约和使用说明

### maven规约

1. maven的依赖声明和属性版本都必须放在grid-app-parent下
2. 子类直接引用父类声明的包，不允许自主定义
3. maven仓库不存在的第三方jar上传到私服

### 业务编码规约

1. 业务代码的书写必须申明业务包

2. 控制层写在模块

   ```
   grid-app-web
   ```

    

   controller包下且命名后缀为

   ```
   *Controller
   ```

   1. 接口路径统一为** /api/模块名/业务名/ **

3. 业务具体实现写在模块`grid-app-service` service包下且命名后缀为`*Service` 和 `*ServiceImpl`

4. 持久层具体实现写在模块`grid-app-dao` dao包下，目前架构兼容spring data jpa和mybatis-plus持久层架构，规定不是特殊要求，使用mybatis-plus,使用repository和mapper包来区分不同的持久层,命名后缀分别为*Repository 和 *Mapper

5. 实体类写在模块`grid-app-pojo` pojo包下,目前规定必须至少定义2个层级：DTO层和DO层,DTO层代表和客户端交互层，DO层代表与数据库交互层，domain和mapper包分别代表spring data jpa和mybatis-plus持久层架构产生的DO,命名后缀为`*DO` 和 `*DTO`

### 异常规约

1. 正常情况业务模块不用关心异常的处理，只关心业务代码
2. 如果想自定义异常，new KeeperException进行统一处理

### 日志规约

1. DEBUG级别下，默认打印方法参数的输入和返回值
2. AuditLog 注解记录自定义需要数据库记录的审计日志
3. logback-spring.xml配置自定义日志输出，线上开启file appender

## 项目下载和运行

- 拉取项目代码 

```
http://47.92.23.16:8098/grid/grid-boot.git    
```

1. 安装jdk、maven、redis等基础环境
2. 切换到grid-app文件夹下

```
# 编译 
mvn compile

# 清除
mvn clean

# 打包
mvn package

# 运行项目
java -jar target/*.jar
```

## 项目部署

1. 打可运行jar，手动上传部署方式（略）
2. devops自动化部署(仅限于开发和测试环境)
   - 自动化容器部署方式（jenkins+sonatqube+maven+docker+gitlab）
     - 找运维配置docker自动化脚本
     - 根据gitlab触发条件，提交代码自动编译打包容器化，上传远程仓库，拉取镜像，启动服务
   - 自动化部署（常规方式）
     - 找运维配置自动化脚本
     - 根据gitlab触发条件，提交代码自动编译打包容器化，上传远程服务，启动服务

- 其他待补充...

## 备注

### 历史事件

- 2021-05-10：按照前后分离模式，基于RBAC模式搭建的接口服务
  - 新增用户 `SysUser` 角色 `SysRole` 菜单`SysMenu` 部门`SysDept`模块
  - spring-security 修改为从`SysUser`中获取用户信息
  - 新增通用查询注解`@Query` 。搭配`QueryGenerator`快速构建mybatisPlus `QueryWrapper`查询条件
- security代码质量检测sonarqube配置，添加swagger全局token验证
- 2020-01-08：项目引入docker脚本，构建jenkins流水线，编写jenkins piline脚本实现docker容器化自动化部署
- 2020-01-05：引入容器的相关配置，集成jib-maven-plugin插件实现基本的容器化部署
- 2020-12-25：构建且测试通过基本的devops，实现基本的自动化部署
- 2020-12-01：构建基本架构环境，发布最初的1.0版本

备注里写清楚每个发布版本更新了什么，等后续版本稳定之后，做为最初的1.0版本