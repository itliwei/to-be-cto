# MySQL

[TOC]

今天起，开始深入学习一下，全球最流行的数据库——MySQL。

我们在学习Redis的时候第一篇，已经讲解了数据库的分类，其中MySQL是属于关系型数据库的一种。关系型数据库的优缺点分析就不在这赘述了。详情可参阅：

### MySQL来源

MySQL是一种开放源代码的关系型数据库管理系统。由于其**体积小**、**速度快**、总体拥有**成本低**，尤其是**开放源码**这一特点，让中小型企业减少不少成本。

![image-20210320093632202](/Users/vince/Library/Application Support/typora-user-images/image-20210320093632202.png)

其实MySQL最初的出发点是用mSQL和他们自己的快速低级例程(ISAM)去连接表格。不管怎样，在经过一些测试后，开发者得出结论：mSQL的速度或灵活性不足以满足要求。这导致了为数据库提供了新的SQL接口，这样，这个API被设计成允许为用于mSQL而写的第三方代码更容易移植到MySQL。

MySQL名称的起源不明。一直以来，我们的基本目录以及大量库和工具均采用了前缀“my”。不过，共同创办人Monty Widenius的女儿名字也叫“My”。时至今日，MySQL名称的起源仍是一个迷。

### MySQL发展史

1996年，mysql 1.0发布

2000年，ISAM升级为MyISAM存储引擎，MySQL开源

2003年，MySQL4.0发布，集成了InnoDB存储引擎

2005年，MySQL5.0发布，提供了视图、存储过程等功能

2008年，MySQK AB公司被Sun公司收购

2009年，Oracle收购了Sun公司，进入了Oracle MySQL时代

2010年，MySQL5.5发布，InnoDB成为了默认的存储引擎

2013年，MySQL5.6发布，相当于MySQL的6.0版本

2015年，MySQL5.7发布，相当于MySQL的7.0版本

2016年，Mysql发布了8.0.0版本

Oracle，大家都知道，是那个很贵很贵的数据库Oracle的公司。被Oracle公司收购后，MySQL会不会也商业化？MySQL的创始人Michael Widenius也有同样的担心，但是不同的是我们只能祈求Oracle有一些道德，而Michael Widenius已经开始主动出击，他以MySQL为基础，开启了另一个分支计划——MariaDB。弱者只能祈求，强者才会出击！

#### MySQL与MariaDB对比

![image-20210320093727370](/Users/vince/Library/Application Support/typora-user-images/image-20210320093727370.png)

二者都是出自一个人的杰作，因其与MySQL保持着高度的兼容性，相应的版本可以直接替换。

1、MariaDB发展趋势和更新频率

毕竟基于MySQL创始人领衔开发的MariaDB数据库，知道MySQL数据库存在的弱点所在，然后提供更好的兼容性和扩展性，我们基本上完全可以将MySQL数据库建议到MariaDB数据库中，而且MariaDB发展速度和升级速度远远优先。

2、MySQL封闭且发展缓慢

由于MySQL在被收购之后更新速度与性能的优化非常的缓慢，而且是闭源的，完全没有Oracle之外的人参与进来，很多需要解决的问题都没有升级进去，反之很多公司虽然也有利用自己开发的分支MYSQL版本。

3、MariaDB的特点和优势

MariaDB基于事务的Maria存储引擎，替换了MySQL的MyISAM存储引擎，它使用了Percona的 XtraDB，InnoDB的变体，MariaDB默认的存储引擎是Aria，Aria可以支持事务，但是默认情况下没有打开事务支持，因为事务支持对性能会有影响。MariaDB是一个采用Maria存储引擎的MySQL分支版本，是由原来 MySQL 的作者Michael Widenius创办的公司所开发的免费开源的数据库。

4、MariaDB与MySQL性能对比

这个直观的区别在于MariaDB能够快速的查询和处理数据，且占用资源相对是少于MySQL数据库的，而且在运行速度、以及支持对 Unicode 的排序问题优于MySQL数据库。


### 什么是MySQL

#### MySQL官方描述

1、**MySQL is a database management system.**

​	MySQL是一个数据库管理系统

2、**MySQL databases are relational.**

​	MySQL数据库是关系型的

3、**MySQL software is Open Source.**

​	MySQL软件是开源的

4、**The MySQL Database Server is very fast, reliable, scalable, and easy to use.**

​	MySQL数据库服务器非常快速、可靠、可扩展且易于使用

5、**MySQL Server works in client/server or embedded systems.**

​	MySQL服务器适用于客户机/服务器或嵌入式系统

6、**A large amount of contributed MySQL software is available.**

​	提供了大量可用的MySQL软件

说了这么多优点，它究竟是如何做到的呢？我们一起来看一下MySQL的官方架构图。

#### MySQL架构图

![image-20210320094542529](/Users/vince/Library/Application Support/typora-user-images/image-20210320094542529.png)

##### 1、连接层

Connectors组件，是MySQL向外提供的交互组件，如java,.net,php等语言可以通过该组件来操作SQL语句，实现与SQL的交互。

##### 2、管理服务组件和工具组件

提供对MySQL的集成管理，如备份(Backup),恢复(Recovery),安全管理(Security)等

##### 3、连接池

负责监听对客户端向MySQL Server端的各种请求，接收请求，转发请求到目标模块。每个成功连接MySQL Server的客户请求都会被创建或分配一个线程，该线程负责客户端与MySQL Server端的通信，接收客户端发送的命令，传递服务端的结果信息等。

##### 4、SQL接口

接收用户SQL命令，如DML,DDL和存储过程等，并将最终结果返回给用户。

##### 5、查询分析器

首先分析SQL命令语法的合法性，并尝试将SQL命令分解成数据结构，若分解失败，则提示SQL语句不合理。

##### 6、优化器

对SQL命令按照标准流程进行优化分析。它使用的是“选取-投影-联接”策略进行查询。

##### 7、缓存

查询缓存，如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。

##### 8、MySQL存储引擎

MySQL存储引擎在MySQL中扮演重要角色，其作比较重要作用，管理表创建，数据检索，索引创建等。

##### 9、文件存储

实际存储MySQL 数据库文件和一些日志文件等的系统，如Linux，Unix,Windows等。

这么复杂而完善的架构设计，那么我们真正执行一条SQL的时候，是如何执行的呢？

### MySQL执行流程

![image-20210320101517025](/Users/vince/Library/Application%20Support/typora-user-images/image-20210320101517025.png)

#### 1、连接

1.1、客户端发起一条Query请求，监听客户端的连接管理模块接收请求；

1.2、将请求转发到连接进/线程模块；

1.3、调用‘用户模块’来进行授权检查；

1.4、通过检查后，‘连接进/线程模块’从‘线程连接池’中取出空闲的被缓存的连接线程和客户端请求对接，如果失败则创建一个新的连接请求。

#### 2、处理

2.1、先查询缓存，检查Query语句是否完全匹配，接着再检查是否具有权限，都成功则直接取数据返回；

2.2、上一步有失败则转交给‘命令解析器’，经过词法分析，语法分析后生成解析树；

2.3、接下来是预处理阶段，处理解析器无法解决的语义，检查权限等，生成新的解析树；

2.4、再转交给对应的模块处理；

2.5、如果是SELECT查询还会经由‘查询优化器’做大量的优化，生成执行计划；

2.6、模块收到请求后，通过‘访问控制模块’检查所连接的用户是否有访问目标表和目标字段的权限；

2.7、有则调用‘表管理模块’，先是查看table cache中是否存在，有则直接对应的表和获取锁，否则重新打开表文件；

2.8、根据表的meta数据，获取表的存储引擎类型等信息，通过接口调用对应的存储引擎处理；

2.9、上述过程中产生数据变化的时候，若打开日志功能，则会记录到相应二进制日志文件中。

#### 3、结果

3.1、Query请求完成后，将结果集返回给‘连接进/线程模块’；

3.2、返回的也可以是相应的状态标识，如成功或失败等；

3.3、‘连接进/线程模块’进行后续的清理工作，并继续等待请求或断开与客户端的连接；

