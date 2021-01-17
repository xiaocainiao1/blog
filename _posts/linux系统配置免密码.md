#                                **linux系统配置免密码**

**一、需求：** 
假设 A 为客户机器，B为目标机, 要达到的目如下： 
A机器ssh登录B机器无需输入密码； 
加密方式选 rsa|dsa均可以，默认dsa 

**二、常用解决方式：** 
1、登录A机器 
2、ssh-keygen -t [rsa|dsa]，将会生成密钥文件和私钥文件 id_rsa,id_rsa.pub或id_dsa,id_dsa.pub 
    例如：ssh-keygen -t rsa -C  "372338206qq.com" 
    -C : 表示注释的意思 
3、将 xxxx.pub的 文件复制到B机器的 .ssh 目录中， 并执行 #cat id_dsa.pub >> ~/.ssh/authorized_keys 
4、操作完成，验证从A机器登录B机器的目标账户，是否还需要密码。 

如果想要B可无密登陆A，操作做个逆向即可。

**三、linux系统配置免密码的快捷方式：**

使用ssh-keygen 创建公钥和密钥。 
使用ssh-copy-id命令传送文件 
ssh-copy-id 把本地主机的公钥复制到远程主机的authorized_keys文件上。
ssh-copy-id 也会给远程主机的用户主目录（home）和~/.ssh, 和~/.ssh/authorized_keys设置合适的权限 。

实例展示：
1：生成密钥
\# ssh-keygen -t rsa -C 
2：把本机的公钥追到192.168.0.1的root目录下的.ssh/authorized_keys文件里

```
# ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.0.1
```

注：如果ssh的端口不是22，可用下面命令

```
# ssh-copy-id -i ~/.ssh/id_rsa.pub  -p 23 root@192.168.0.1
```

**四、ssh启动和停止**

停止:   service sshd stop

启动：service sshd start

调试： /usr/sbin/sshd -d

**五、遇到的问题**

1. 首先保证ssh服务是启动的

2. 修改配置文件

   ```
   vim /etc/ssh/sshd_config
   RSAAuthentication yes
   PubkeyAuthentication yes
   AuthorizedKeysFile    %h/.ssh/authorized_keys
   ```

3.  

**六、日志查看**

```
tailf /var/log/secure
ssh -v pre1
```

**七、ssh无密码登陆 Authentication refused: bad ownership or modes for directory /root**

  

```
解决方法：

chmod  0700 /root -R

or chown root.root root
```



注意：
**要保证.ssh和authorized_keys都只有用户自己有写权限。否则验证无效。**

```
//用户权限
chmod 700 /home/username |   /root
//.ssh文件夹权限
chmod 700 ~/.ssh/x	
// ~/.ssh/authorized_keys 文件权限
chmod 600 ~/.ssh/authorized_keys
```

