​                                     **访问docker desktop创建的Hyper-v虚拟机DockerDesktopVM**

平常我们部署kubernetes ,一般都是先安装个linux操作系统,不管是centos或者ubuntu或者其他,都是我们手工操作的.安装好的这个操作系统都开启了SSH,能够远程登录访问. 但是docker desktop 为我们自动创建了一个Hyper-v的虚拟机,你打开虚拟机管理器会看到,这个虚拟机的名字叫做DockerDesktopVM, 从Hyper-v的管理界面无法连接到这个虚拟机,完全就是一个黑箱,甚至你发现这个虚拟机连一个网卡都没配置.实在是太黑暗了.

通过下面的操作可以连接进入.

在你的cmd或者powershell中分别执行下面三行命令.

```
docker run --privileged -it -v /var/run/docker.sock:/var/run/docker.sock jongallant/ubuntu-docker-client 
docker run --net=host --ipc=host --uts=host --pid=host -it --security-opt=seccomp=unconfined --privileged --rm -v /:/host alpine /bin/sh
chroot /host
```

这样就连接进来了.

wget www.google.com 一下.发现又链接不上了. ping 你的网卡ip又是通的,你的代理也正常,为啥又不行了呢. 

先不管它, 在DockerDesktopVM虚拟机中再配置一次代理. 参见https://www.cnblogs.com/worldinmyeyes/p/12319405.html

完成.

uname -a 一下,你会发现,这个linux版本为linuxkit, 是Docker新发布的一个用于为容器构建安全、便携、可移植操作系统的工具包。