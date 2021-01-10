#                                                Markdown 编辑器设置字体大小颜色



Markdown是一种可以使用普通文本编辑器编写的标记语言，通过类似HTML的标记语法，它可以使普通文本内容具有一定的格式。但是它本身是不支持修改字体、字号与颜色等功能的！

  CSDN-markdown编辑器是其衍生版本，扩展了Markdown的功能（如表格、脚注、内嵌HTML等等）！对，就是内嵌HTML，接下来要讲的功能就需要使用内嵌HTML的方法来实现。

# 一、更改字体、大小、颜色

```
<font face="黑体">我是黑体字</font>
<font face="微软雅黑">我是微软雅黑</font>
<font face="STCAIYUN">我是华文彩云</font>
<font color=red>我是红色</font>
<font color=#008000>我是绿色</font>
<font color=Blue>我是蓝色</font>
<font size=5>我是尺寸</font>
<font face="黑体" color=green size=5>我是黑体，绿色，尺寸为5</font>
```


效果如下： 

<font face="黑体">我是黑体字</font>
<font face="微软雅黑">我是微软雅黑</font>
<font face="STCAIYUN">我是华文彩云</font>
<font color=red>我是红色</font>
<font color=#008000>我是绿色</font>
<font color=Blue>我是蓝色</font>
<font size=5>我是尺寸</font>
<font face="黑体" color=green size=5>我是黑体，绿色，尺寸为5</font>

------

# 二、为文字添加背景色

由于 style 标签和标签的 style 属性不被支持，所以这里只能是借助 table, tr, td 等表格标签的 bgcolor 属性来实现背景色。故这里对于文字背景色的设置，只是将那一整行看作一个表格，更改了那个格子的背景色（bgcolor）

Markdown语法

```
<table><tr><td bgcolor=yellow>背景色yellow</td></tr></table>
```

效果如下

<table><tr><td bgcolor=yellow>背景色yellow</td></tr></table>

------

# 三、设置图片大小

## 1、设置设置图片百分比

Markdown语法

```
<img src="http://pic11.photophoto.cn/20090626/0036036341009653_b.jpg" width="50%" height="50%">
```

效果如下 
<img src="http://pic11.photophoto.cn/20090626/0036036341009653_b.jpg" width="50%" height="50%">

## 2、设置图片大小

Markdown语法

```
<img src="http://pic11.photophoto.cn/20090626/0036036341009653_b.jpg" width="251" height="350" >
```

效果如下 
<img src="http://pic11.photophoto.cn/20090626/0036036341009653_b.jpg" width="251" height="350" >

## 3、设置图片居中

Markdown语法

```
<div align=right><img src="http://pic11.photophoto.cn/20090626/0036036341009653_b.jpg" width="50%" height="50%"></div>
```

效果如下

<div align=right><img src="http://pic11.photophoto.cn/20090626/0036036341009653_b.jpg" width="50%" height="50%"></div>

# 参考链接

[RGB颜色对照表](https://blog.csdn.net/heimu24/article/details/81192697)

[Markdown 语法大全 包括设置字体 颜色](https://blog.csdn.net/qcx321/article/details/53780672)

[Markdown: Basics （快速入门）](https://www.appinn.com/markdown/basic.html)

[CSDN-markdown 文字样式设置（字体, 大小, 颜色, 高亮底色）](https://blog.csdn.net/thither_shore/article/details/52181464)

# 

