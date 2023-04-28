##                                         k8s笔记

# 1.k8s安装之Linux centos7升级内核到4.18以上

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
#查看最新版内核： yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
#安装最新版：
yum --enablerepo=elrepo-kernel install kernel-ml kernel-ml-devel –y
#查看当前可用内核版本：
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
#选择最新内核版本，0代表查看当前可用内核版本列表的左侧索引号
grub2-set-default 0
#生成grub文件
grub2-mkconfig -o /boot/grub2/grub.cfg
#重启linux
reboot
#查看内核
uname -r
```



