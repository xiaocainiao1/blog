#                      linux运维常用命令

# 内存详细信息查看

```
linux查看内存信息命令: cat /proc/meminfo
free命令是一个快速查看内存使用情况的方法，它是对 /proc/meminfo 收集到的信息的一个概述
1. 内存

total:内存总数

used：已使用的内存数

   free:空闲的内存数

shared:当前已废弃不用

buffers:系统分配但未被使用的缓冲区

cached:系统分配但未被使用的缓存

(buffers和cached区别：A buffer is something that has yet to be “written” to disk. A cache is something that has been “read” from the disk and stored for later use（缓冲区还没有被写入磁盘。 缓存是从磁盘“读取”并存储以备后用的）)

2. 程序已用内存数：

-(buffers/cached):used 第一部分mem行 used-buffers-cached (反应的被程序实实在在吃掉的内存)

程序可用内存数

+(buffers/cached):free 第一部分 mem行 free+buffers+cached （可以挪用的内存总数	
```

# 查看CPU信息

```
查看CPU个数:cat /proc/cpuinfo | grep "physical id" | uniq | wc -l  --uniq命令：删除重复行;wc –l命令：统计行数
查看CPU核数:cat /proc/cpuinfo | grep "cpu cores" | uniq
```



# linux查看磁盘使用情况命令

#### 一、查看磁盘文件的可用空间

###### 1、df命令简介

linux中df命令的功能是用来检查linux服务器的文件系统的磁盘空间占用情况。可以利用该命令来获取硬盘被占用了多少空间，目前还剩下多少空间等信息。

###### 2、命令格式：

df [选项] [文件]

###### 3、使用示例

查看当前目录磁盘使用情况：



```undefined
df -h
```

查看指定目录磁盘使用情况：



```kotlin
df -h /data/
```

#### 二、具体查看文件夹的占用情况

###### 1、du命令简介

Linux中du命令用于显示目录或文件的大小，du会显示指定的目录或文件所占用的磁盘空间。

###### 2、命令格式：

```
du [-abcDhHklmsSx][-L <符号连接>][-X <文件>][--block-size][--exclude=<目录或文件>][--max-depth=<目录层数>][--help][--version][目录或文件]
```



###### 3、使用示例

查看当前目录每个文件夹的情况：



```swift
du --max-depth=1 -h 
```

查看指定目录每个文件夹的情况：



```kotlin
du --max-depth=1 -h  /data/
```

计算指定文件夹大小



```kotlin
du -sh /data/
```

# 查看端口占用情况

```
netstat -tunlp
```

