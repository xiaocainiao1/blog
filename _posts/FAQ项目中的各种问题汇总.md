#                                    **FAQ项目中的各种问题汇总**

###### 注册中心报无法注册缓存找不到

 

```
java.lang.IllegalArgumentException: Host name may not be null
```

这个原因是注册中心自己注册了自己，禁止注册自己就好了

```
	eureka:
    instance:
        prefer-ip-address: true
    client:
        enabled: true
        fetch-registry: false
        register-with-eureka: false
```



###### 指定外部bootstrap不生效 --spring.cloud.bootstrap不生效

​       初步认定是版本问题，springboot2.X的bug，升级版本有待验证

###### 配置中心配置fetch-registry: false,register-with-eureka: false不生效，还是会读取jar包里的，有待验证原因

```
经过多方调试，并非不生效，而是在application-seata-dev.yml里的注册中心里的配置错误，还是不认真引起的
```



- fetch-registry: false,register-with-eureka: false即使填写了，

  ```
  service-url:
              defaultZone: http://admin:${spring.security.user.password}@localhost:8761/eureka/
  ```

  如何填写的不是localhost,也还是尝试注册的，所以如果不想用集群，一定要改为localhost

- was unable to refresh its cache! status = Cannot execute request on any known server

> 出现这种错误是因为:
>
> ```
> Eureka服务注册中心也会将自己作为客户端来尝试注册它自己，所以我们需要禁用它的客户端注册行为。
> ```
>
> 在 yml中设置
>
> ```
> eureka.client.register-with-eureka=false 
> eureka.client.fetch-registry=false
> ```
>
> 但在服务端是要这是为false的，在客户端需要设置为true的，当在客户端设置为true之后若还是报这个错误，说明配置有问题。
>
> 有一个很坑人的是
>
> ```
> eureka.client.serviceUrl.defaultZone= http://localhost:8761/eureka/
> ```
>
> 这段代码中的url中的eureke不能换成任何其他的（客户端和服务端都不能换，即使换成一样的也不行）
>
>  
>
> 坑！！！

###### application-dev.yml  放在config目录下，会引起配置中心错乱

###### 前端webpack一直编译不成功，问题引起的原因就是网络问题，下载的东西少东西，上传自己本地的上去就没问题了

###### windows bat 命令闪退，从命令行启动能看到闪退的原因

###### 修改打印的神注释

  source文件夹下banner.txt里修改

###### rm: cannot remove `xxx’: Operation not permitted的解决方法

```
第一步：查看文件属性
lsattr xxx  查看文件属性(xxx为文件名)
1
看到的情况

-----a-------或者-----i-------
1
第二步：去除属性
chattr -a xxx 或者 chattr -i xxx
1
第三步：删除文件
rm -rf xxx
1
注意：
如果在执行第二步的时候报错（没有着命令），那么需要重新下载chattr的依赖包
我的操作系统是centos7

yum -y reinstall e2fsprogs-1.42.9-16.el7.x86_64
1
echo $PATH
```

###### springboot 服务运行，service里没有springboot启动类的配置

.idea/workspace.xml 添加

```
<component name="RunDashboard">
  <option name="configurationTypes">
      <set>
        <option value="SpringBootApplicationConfigurationType" />
      </set>
    </option>
    <option name="ruleStates">
      <list>
        <RuleState>
          <option name="name" value="ConfigurationTypeDashboardGroupingRule" />
        </RuleState>
        <RuleState>
          <option name="name" value="StatusDashboardGroupingRule" />
        </RuleState>
      </list>
    </option>
  </component>
```

