#                             java开发经验分享

# 一、 编码

1. 约束自己，规范编码习惯

充足的代码注释、标准缩进的格式、注意命名规范。参考《开发规约》

"看上去"专业能促进代码质量。越是难看的代码，在它的演化过程中会越来越差。因为当你看到你要bugfix的代码很乱，那么在bugfix的时候一般也会草草了事。一个方法有400行，在修改这个方法的时候大家不会在意再加上几十行代码。因为它看起来很差，它就是很差，我没必要美化它。反过来说，如果待改的代码很整洁很规范，那么修改者也会比较小心。

2. 避免冗长的方法和类

应将方法设计成简要的、功能性单元，用它描述和实现一个不连续的类接口部分。理想情况下，方法应简明扼要。若长度很大，可考虑通过某种方式将其分割成较短的几个方法。这样做也便于类内代码的重复使用（有些时候，方法必须非常大，但它们仍应只做同样的一件事情）。

3. 不要向标准输出写无用数据

System.out.println(); 此语句在程序中随处可见，都是在调试时使用的，在程序正式运行时也没有将其去掉，结果就是日志中有大量的无用数据，不仅使得日志不易分析，也增加了系统的开销。

4. 调用方法前注意参数校验，判断参数是否为空或无意义的值

5. 使用对象前，检查对象是否为空

if(names!=null&&names.length>0){

String[] nameArray = names.split(‘,’);

}

if(userEn!=null){

String name = userEn.getName();

}

6. 避免过多过常的创建java对象

尽量避免在经常调用的方法、循环中new对象，由于系统不仅要花费时间来创建对象，而且还要花时间对这些对象进行垃圾回收和处理，在我们可以控制的范围内，最大限度的重用对象，最好能用基本的数据类型或数组来替代对象。

7. 尽量避免随意使用类成员变量

当某个对象被定义为static变量所引用，那么gc通常是不会回收这个对象所占有的内存的。此时类成员变量的生命周期与类同步，如果类不卸载，那么该对象会常驻内存，直到程序终止

8. 减少对变量的重复计算

如

for(int i=0;i<list.size();i++)

应该改为

for(int i=0,len=list.size();i<len;i++)

并且在循环中应该避免使用复杂的表达式，在循环中，循环条件会被反复计算，如果不使用复杂表达式，而使循环条件值不变的话，程序将会运行的更快

9. 避免不必要的创建对象

如

A a = new A();

if(i==1){

list.add(a);

}

应该改为

if(i==1){

A a = new A();

list.add(a);

}

10. 原则上循环里面不要声明对象，一律在循环外面声明

for(int i=0;i<size;i++){

String title = “标题”;

}

改为

String title = null;

for(int i=0;i<size;i++){

title = “标题”；

}

11. 尽量在finally块中释放资源

程序中使用到的资源应当被释放，以避免资源泄漏。这最好在finally块中去做。不管程序执行的结果如何，finally块总是会执行的，以确保资源的正确关闭。

12. 使用StringBuilder和StringBuffer进行字符串连接

StringBuffer提供了同步机制，所以并发线程访问是线程安全的，适合多线程。

StringBuilder没有提同步机制，所以线程不安全，适合单线程，但如果是单线程的话，要比StringBuffer快。

13. 遍历HashMap使用entrySet

当需要遍历HashMap的时候，请尽量使用entrySet，而不要用keySet，entrySet的效率要比keySet高，实际上使用entrySet是只需要遍历一次hash，即将key和value的映射关系放入到entry中，再取之；而keySet需要两次遍历hash，第一次取所有的key，第二次用key去取出对应的value。

Iterator iter = hashMap.entrySet().iterator();

while (iter.hasNext()) {

Map.Entry entry = (Map.Entry) iter.next();

String key = String.valueOf(entry.getKey());

String val = String.valueOf(entry.getValue());

}

14. 尽量缓存经常使用的对象

尽可能将经常使用的对象进行缓存，可以使用数组，或HashMap的容器来进行缓存，但这种方式可能导致系统占用过多的缓存，性能下降。

15. 使用统一的工具类

使用hanwebcommon.jar中的通用方法

使用项目中已经存在的工具类，不要重复创造功能近似的类和方法，如果必要可进行扩展

如：接收参数使用Convert.getParameter(request, 参数名);

16. 减少不必要的空格和空行

17. java代码中不要出现黄色警告。注释或删除未使用的变量；保存时去掉多余的import；…

18. 前台接收Stirng类型参数，要进行跨站脚本和sql注入过滤

Convert.getParameter(request,"keyword","",true,true);

19. 不要在controller中实现业务逻辑，放到service类中去完成

分层设计实现了软件之间的解耦；便于进行分工；便于维护；提高软件组件的重用；

20. 避免在循环体中使用try-catch 块,最好在循环体外使用try--catch 块以提高系统性

21. oracle大字段操作

先插入一个空的clob类型 empty_clob()，然后再单独更新clob字段

InsertSql insql = new InsertSql( strTableName ); 

  insql.addString("vc_name", name);

  if (("oracle").equals(SysInit.getM_strDB_Type())){

   insql.addClob("vc_adress");

   insql.addClob("vc_path");

  }else{

​    insql.addString("vc_adress", address);

​    insql.addString("vc_path", path);

}

boolean bl = Manager.doExcute(strAppID , insql.getSql());

if(bl){

if (("oracle").equals(SysInit.getM_strDB_Type())){

String[] strFieldValue = {address, path};

String[] strFieldName ={"vc_adress","vc_path"};

Manager.doClob(strAppID, strFieldName, strFieldValue, strTableName, " WHERE i_id = " + getMaxId());

}

}

22. 使用统一的<!DOCTYPE>，保证不同浏览器下的页面兼容

建议使用：

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

23. HTML结构要完整、正确

标准的HTML文档结构：
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html>

<head>

<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />

<title>Insert title here</title>

……

</head>

<body>

……

</body>

</html>

其它：

<ul>

<li>……</li>

<li>……</li>

</ul>

<table>

<tr>

<td>……</td>

</tr>

</table>

24. HTML标签要完整

标签名称及属性统一使用小写，标签要成对出现，如：

<div id=”search”>………</div>

不用于包含内容的标签，可在标签结尾使用“/”标记结束，如：

<input type=”text” name=”username” value=”tony” />

<br/> 

25. 标签属性值必须用双引号包住

26. HTML代码使用标准缩进

27. 脚本每一条语句都要以分号结尾

28. 具有独特性、不需要重复使用的样式，使用内嵌样式：

<div style=”title”>标题<div>

能够重复使用的样式，在样式表中定义：

<li class=”menu”>菜单</li>

页面内使用的样式，使用内嵌样式表：

<style type="text/css">

.menu{

color:black;

font-size:13px;

}

</style>

多个页面公用的样式使用链入外部样式表：

<link href="../global.css" rel="stylesheet" type="text/css" />

29. 页面内使用的脚本函数，在head中定义：

<script type=”text/javascript”>

function checkForm(){

……

}

</script>

重复使用的脚本函数或代码较多的脚本，写到外本脚本文件中，在head中链接：

<script type=”text/javascript” src=”../checkform.js” />

不影响页面本身展示的脚本可考虑放在body结束的位置调用，如广告代码：

……

<script type=”text/javascript” src=”../adv.js” />

</body>

30. 风格统一，保存时要对代码进行格式化，Ctrl+Shift+F

31. 养成程序优化的意识

现在我们经常遇到的一个问题是，程序在开发阶段，执行的完全正常，找测试人员测试也没问题，但是一到上线后，性能马上出问题了，运行速度像蜗牛，客 户不堪忍受，为什么？简单的说，开发人员自测，估计也就几条数据，测试人员测试，估计也就几十上百条数据，一般的程序代码，在这个数量级，性能瓶颈根本就 表现不出来。但是上线后，客户的数据一般都上升到10000级别的，如果程序代码中没有做到细微之处都很严谨的话，问题就马上暴露出来了。

# 二、 数据库

1. SQL语句中保留字、函数名要大写，表明、字段名全部小写

如：SELECT vc_name,vc_sex,i_age FROM user WHERE i_id = 100 AND i_type = 2

2. 使用标准SQL语句，防止数据库兼容问题

3. 循环里面（包括循环调用方法内）避免过多操作数据库

4. 选择最有效率的表名顺序

ORACLE 的解析器按照从右到左的顺序处理FROM子句中的表名，FROM子句中写在最后的表(基础表 driving table)将被最先处理，在FROM子句中包含多个表的情况下,必须选择记录条数最少的表作为基础表。如果有3个以上的表连接查询, 那就需要选择交叉表(intersection table)作为基础表, 交叉表是指那个被其他表所引用的表

5. 注意WHERE子句中的连接顺序

ORACLE采用自下而上的顺序解析WHERE子句,根据这个原理,表之间的连接必须写在其他WHERE条件之前, 那些可以过滤掉最大数量记录的条件必须写在WHERE子句的末尾

6. SELECT子句中避免使用*

ORACLE在解析的过程中, 会将'*' 依次转换成所有的列名, 这个工作是通过查询数据字典完成的, 这意味着将耗费更多的时间

7. 减少访问数据库的次数，尽量批量操作数据库，如批量删除

ORACLE在内部执行了许多工作: 解析SQL语句, 估算索引的利用率, 绑定变量 , 读数据块等

8. 避免在WHERE子句中使用in，not in，or 或者having

可以使用 exist 和not exist代替 in和not in

9. 用WHERE子句替换HAVING子句

避免使用HAVING子句, HAVING 只会在检索出所有记录之后才对结果集进行过滤. 这个处理需要排序,总计等操作. 如果能通过WHERE子句限制记录的数目,那就能减少这方面的开销。

10. 用好数据库事务

事务是指作为单个逻辑工作单元执行的一系列操作。通过将一组相关操作组合为一个要么全部成功要么全部失败的单元，可以简化错误恢复并使应用程序更加可靠。

Transaction ts = null;

try{

ts = new Transaction(appId);

…

…

boolean bl = ts.execute(sql);

if(bl){

bl = ts.commit();

}

if(!bl){

throw new Exception();

}

}catch(Exception e){

if(ts!=null)

ts.roolback();

bl = false;

}

11. 注意SQL执行效率，考虑单表记录10W以上的运行效果

setup中，日志显示级别为警告以上时，执行时间超过300ms的SQL语句，会在日志中输出warning

12. 索引

参考：[数据库性能优化](http://www.cnblogs.com/xhp956614463/p/5342180.html)

# 三、 项目开发

1. 需求：

1) 需求最终需要开发人员在产品中实现，开发不合理的设计会浪费时间，开发技术无法实现的设计带来最大的痛苦：失败。所以，开发人员要重视需求以及需求评审，提出自己能够想到的所有异议；

2) 开发人员不但要做好需求分析，还要做出精确的估计。因为编码工作保质保量的按时完成需要多方的准备工作，技术难点需要进行充分的技术预言，不熟悉的依赖平台或类库要进行熟悉；

2. 计划：一栋楼很难估算重量，但是一块砖头可以精确估算重量。一个项目的时间很难准确的估计，但把项目开发划分为不能再进行分割的模块功能点，对每个点的估计是可以更精准的估时的，由此由上至下，由下至上，可以得出近乎准确的开发时间。

3. 设计：

1) 一图胜万言，模块结构以及流程等很难用用文字描述，即使用文字描述出来也很难看懂，所以在设计中，要善用用图；

2) 详细设计过程中有思考的痛苦，繁琐的痛苦，但是绕过这些痛苦，编码期间将会面临更大的痛苦，要以快乐的心态面对。

4. 编码：

1) 对于一个实现可以有很多解决方案，花些时间精力选取你认为最好的解决方案可以总体上提高工作成效，往往还可以得到用户更好的体验效果；

2) 细致认真严谨的工作即是对工作负责，更是对自己负责，让这些成为习惯。任何一次，任何时候所进行的编码工作，在逻辑、风格、简单有效等方面都尽可能的做到最好，既能更好为公司实现价值，同时更有利自己在技能，岗位的进步；

3) 简单是美，在有效的前提下，越是简单的处理方法越是珍贵的，代码编写也是，简单的代码便于理解维护，同时不容易产生错误

4) 慎做改动，当然不是说不做改动或不鼓励改动，而是不做仓促、草率的代码改动。没有洞察全局，考虑全面，而仓促进行的改动往往没有达到改动的目的却带来了其他问题

5) 模块的性能不是减少一行或几行执行代码所能提高的，性能的优化首先是从算法上考虑，降低时间复杂度，然后从执行逻辑入手，减少循环执行代码的执行次数

6) 关键地方要打印日志输出到文件中，在运行过程中不断检查日志，发现任何异常都要检查原因并修改

5. 测试：

1) 事出有因，任何bug都是由于代码的疏漏造成的，利用排除法或跟踪调试代码等方法找到疏漏所在；

2) 遇到自身模块相关问题首先检查自己，相互推诿只会浪费时间以及减弱在其他同事对你的信任；

3) 站的高看得远，不同的视角有不同的风景。遇到比较难解决的问题而苦苦没有思路时，转换思路或把问题的考虑范围放的更广一点，往往可以找到解决方案

4) 功能提交测试前或bug修复提交验证前，开发人员都要自己详细的测试一下，验证无误再提交。

6. 其它：

1) 善于及时的沟通。在项目的整个流程过程中，遇到他人的问题或自己解决不了的问题，切忌堆在自己心里，要及时找问题解决方进行沟通，寻求解决方案

2) 善于发现并学习别人的长处。作为开发人员，我们在追求接近完美的同时，也需要学会欣赏别人的长处，发现别人的优点，并学习别人的优点，转化为自己的潜质，这样，我们才可以进步的更快，更全面

3) 善于帮助他人解决问题以及进行知识经验的分享，更有利于自己的提高，同时还可以获得他人的尊重

# 四、 关于测试

1. 在整个项目计划中，测试时间安排的合理性，对测试阶段的情况应作充分预计，不可为了赶发布点而忽略质量。

2. 务必清楚产品包、更新包、bug包的提交规范。具体请参照《开发规范手册》。不要出现测试过程中提交多个bug包，或者提交安装包给测试人员更新的情况。

3. 提包时请先检查，路径是否正确、文件是否完整、配置文件是否应该提交，源代码修改记录描述是否完善。之前经常出现，提包不检查就直接提交来测试，导致出错后环境不断地还原或者重建新的环境。CVS使用不熟悉，提交文件反复出错。

4. 产品质量需要严格控制，自己承认是问题的情况下，请不要试图和测试商谈希望可以不被追究。

5. 请注意提高修改bug的质量，目前，修改一个bug而引发更多的bug的情况特别多。

6. bug修改完成后，在提交测试前请自己先验证通过后再提交，请不要修改完成后不验证就直接提交给测试人员，防止导致bug被反复reopen的情况。

7. 原则上不允许不通过CVS而直接提交文件给测试人员调试问题、寻找原因。如果确有必要，测试人员可以协助调试，但次数不宜过多，防止测试环境版本难以控制。

8. bug是否存在的衡量标准以测试环境为准，不建议出现“我这边是好的”这样的解释。

9. 修改bug时请修改完整，可能会有多个小问题提交在一个BUG里面，BUG修改时多个小问题的地方均需修改。

10. 更新文件必须通过配置发布，无论是否经过测试都不允许直接发给项目或直接更新客户服务器