#                            Linux离线安装docker 教程

**前言**

有时候会遇到服务器不能联网的情况，这样就没法用yum安装软件，docker也是如此，针对这种情况，总结了一下离线安装docker的步骤，分享给大家，安装前确定linux内核版本，要求CentenOS 7 内核3.1以上，内核查看命令： uname -a

**1. 准备docker离线包**

下载地址为：http://25.213.30.100:3000/cxpt-aip/tools/outcomes/download/1a7f989f-075d-40be-a7c5-2cc484a03290

下载需要安装的docker版本，我此次下载的是

```
docker-18.06.1-ce.tgz版本
```

**2. 准备docker.service 系统配置文件**

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
 
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=multi-user.target
```

**3. 准备安装脚本和卸载脚本**

安装脚本 `install.sh`

```
#!/bin/sh
echo '解压tar包...'
tar -xvf $1
echo '将docker目录移到/usr/bin目录下...'
cp docker/* /usr/bin/
echo '将docker.service 移到/etc/systemd/system/ 目录...'
cp docker.service /etc/systemd/system/
echo '添加文件权限...'
chmod +x /etc/systemd/system/docker.service
echo '重新加载配置文件...'
systemctl daemon-reload
echo '启动docker...'
systemctl start docker
echo '设置开机自启...'
systemctl enable docker.service
echo 'docker安装成功...'
docker -v
```

**卸载脚本 uninstall.sh**

```
#!/bin/sh
echo '删除docker.service...'
rm -f /etc/systemd/system/docker.service
echo '删除docker文件...'
rm -rf /usr/bin/docker*
echo '重新加载配置文件'
systemctl daemon-reload
echo '卸载成功...'
```

**4. 安装**

4.1 此时目录为：（只需要关注docker-18.06.1-ce.tgz、docker.service、install.sh、uninstall.sh即可）

![img](https://img.jbzj.com/file_images/article/201908/2019080709531410.png)

4.2 执行脚本 `sh install.sh docker-18.06.1-ce.tgz `

执行过程如下：

![img](https://img.jbzj.com/file_images/article/201908/2019080709531411.png)

待脚本执行完毕后，执行 `docker -v `

发现此时docker已安装成功，可以用` docker --help `查看docker命令，从现在开始你就可以自己安装image和container了

4.3 如果你想卸载docker，此时执行脚本` sh uninstall.sh `即可，执行过程如下：

![img](https://img.jbzj.com/file_images/article/201908/2019080709531512.png)

此时输入 `docker -v` ，发现docker已经卸载