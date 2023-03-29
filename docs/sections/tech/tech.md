## 一、技术架构

在企业级知识图谱应用中，数据源主要分为三类：授权数据，公开数据，三方数据，业务数据

- 授权数据：用户允许我们去抓取的数据，比如邮箱里的数据
- 公开数据：网上披露的一些黑名单等等公开数据
- 三方数据：别人给的一些数据
- 业务数据：用户填到系统的一些数据，比如身份证号，手机号，家庭住址工作等等数据。

对于授权数据和公开数据，我们主要通过爬虫的方式获取，三方数据主要通过接口的方式API来进行获取，业务数据直接写到mysql当中的数据，其中这些数据都会存储到mysql当中，相当于一个数据持久化的过程。

Bin log:是mysql自带的一个机制，当mysql每次做出一些数据更新的动作，就会把这条数据写到Binlogo当中，主要的作用主要是实时处理和离线增量导入框架的一个基本数据的提供，主要通过canal server中间键去监控Binlog，通过Canal Server 把数据发向Canal Client,完成数据的传输，拿到数据之后，通过kafka的生产者将数据继续向下传输给kafka，（kafka是一个消息队列），然后再到kafka的消费者消费这些数据，然后把数据通过调用Neo4j的Driver,最终写到Neo4j的数据库中。

数据初始化：最开始没有数据，要想把业务数据直接都写到Neo4j中的做法，首先用mysql的Connector，加上APOC组件，就能够把我们整个数据写到Neo4j中。

数据写入之后，会通过一些规则引擎，提供一个微服务，Restful接口向外提供服务给一些业务系统，然后会把Neo4j当中的数据写入spark Graphx当中，在其之上做一些算法相关的工作。

最终的spark Graphx也有一些模型和算法的规则，也是通过微服务的形式，最后向外面提供服务。

![image-20210112164211690](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112164211.png)

## 二、 Neo4j

数据库的划分

![image-20210112224520058](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112224805.png)

### 1. 安装及部署

1. 进入官网：https://neo4j.com/download/?ref=try-neo4j-lp，点击下载 Neo4j server

![image-20210112225612614](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112225612.png)

2. 然后找到对应自己的操作系统点击下载对应的社区版即可

![image-20210112225721772](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112225721.png)

3. 下载完成后解压到主目录，“D:\Program Files\neo4j-community-4.2.”。

Neo4j应用程序有如下主要的目录结构：

- bin目录：用于存储Neo4j的可执行程序；
- conf目录：用于控制Neo4j启动的配置文件；
- data目录：用于存储核心数据库文件；
- plugins目录：用于存储Neo4j的插件；

4. 创建ne04j的环境变量

创建主目录环境变量NEO4J_HOME，并把主目录设置为变量值。

### 2. 网络连接配置

neo4j支持三种网络协议，默认情况下，不需要配置就可以在本地直接运行。

1. Neo4j支持三种网络协议（Protocol）

Neo4j支持三种网络协议（Protocol），分别是Bolt，HTTP和HTTPS，默认的连接器配置有三种，为了使用这三个端口，需要在Windows防火墙中创建Inbound Rules，允许通过端口7687，7474和7473访问本机。

![img](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112234601.png)

2. 连接器的可选属性

![img](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112234630.png)

listen_address：设置Neo4j监听的链接，由两部分组成：IP地址和端口号（Port）组成，格式是：<ip-address>:<port-number>

3. 设置默认的监听地址

设置默认的网络监听的IP地址，该默认地址用于设置三个网络协议（Bolt，HTTP和HTTPs）的监听地址，即设置网络协议的属性：listen_address地址。在默认情况下，Neo4j只允许本地主机（localhost）访问，要想通过网络远程访问Neo4j数据库，需要修改监听地址为 0.0.0.0，这样设置之后，就能允许远程主机的访问。

```text
# With default configuration Neo4j only accepts local connections.
# To accept non-local connections, uncomment this line:
dbms.connectors.default_listen_address=0.0.0.0
```

4. 分别设置各个网络协议的监听地址和端口

HTTP链接器默认的端口号是7474，Bolt链接器默认的端口号是7687，必须在Windows 防火墙中允许远程主机访问这些端口号。

```text
# Bolt connector
dbms.connector.bolt.enabled=true
#dbms.connector.bolt.tls_level=OPTIONAL
#dbms.connector.bolt.listen_address=0.0.0.0:7687

# HTTP Connector. There must be exactly one HTTP connector.
dbms.connector.http.enabled=true
#dbms.connector.http.listen_address=0.0.0.0:7474

# HTTPS Connector. There can be zero or one HTTPS connectors.
#dbms.connector.https.enabled=true
#dbms.connector.https.listen_address=0.0.0.0:7473
```

### 3. 启动Neo4j程序

1. 通过控制台安装Neo4j程序

点击组合键：Windows+R，输入cmd，启动DOS命令行窗口，切换到主目录，以管理员身份运行，输入以下命令，通过控制台启用neo4j程序

```bash
neo4j.bat console
```

如果出现以下提示，说明neo4j已经开始运行

![img](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112231122.png)

2. 把Neo4j安装为服务（Windows Services）

安装和卸载服务：

```bash
bin\neo4j install-service
bin\neo4j uninstall-service
```

启动服务，停止服务，重启服务和查询服务的状态：

```bash
bin\neo4j start
bin\neo4j stop
bin\neo4j restart
bin\neo4j status
```

3. 打开浏览器输入地址http://localhost:7474  默认跳转到  http://localhost:7474/browser

默认用户名为neo4j，默认密码为neo4j，第一次成功connect到Neo4j服务器之后，需要重置密码。例如我改成了123456。

![image-20210112234247725](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112234247.png)

**卸载：**

当我们需要卸载它的时候，会发现没有卸载的软件。

此时以管理员身份进入cmd，进入到./bin目录下，执行以下命令 ，就会自动停止服务并卸载。

注意一定要在管理员身份下进入cmd

win10下的操作是：开始菜单搜索栏中搜索cmd，右键，以管理员身份运行。

```bash
neo4j uninstall-service
```

### 4. 使用

![image-20210112185336155](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112225019.png)



![image-20210112185340776](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112225026.png)



## 三、APOC

APOC was also the first bundled A Package Of Component for Neo4j .

### 1. 下载

下载jar包：https://github.com/neo4j-contrib/neo4j-apoc-procedures/releases

mysql-connector-java下载地址：http://mvnrepository.com/artifact/mysql/mysql-connector-java

### 2. 安装

把jar包放在neo4j安装目录的plugins文件夹下

### 3. 配置

1. 找到neo4j安装目录下conf文件夹里的neo4j.conf并打开

   在neo4j.conf文件下加上：

   ![image-20210119205452624](https://gitee.com/zgf1366/pic_store/raw/master/img/20210119205452.png)

```text
dbms.security.procedures.unrestricted=apoc.*
# 增加页缓存到至少4G，推荐20G:

dbms.memory.pagecache.size=4g
# JVM堆保存留内存从1G起，最大4G:

dbms.memory.heap.initial_size=1g
dbms.memory.heap.max_size=4g
```

2. 重启Neo4j服务

3. 在可视化界面运行：return apoc.version()，如果出现对应的版本号，证明安装成功

> 注意：neo4j 和 apoc 版本必须对应起来，否则启动Neo4j有可能出现无法连接这种情况                                     例如Neo4j版本为3.4.6 APOC为3.4.0                                                                  

### 4. 功能

#### 4.1 Text and Lookup Indexes (文本和素引查找）

提供素引查询、管理、全文图标和搜索等功能

#### 4.2 Utility Functions(实用函数）

域名搜取、时间和日期、数字格式转换等功能

#### 4.3 Graph Algorithms(图算法）

社区检测、PageRank、中心算法等

#### 4.4 Spatial(空间函数)

地理编码、位置计算、空间与时间搜索等

#### 4.5 Data Integration（数据集成）


JSON、JDBC、csv等格式数据加载

#### 4.6 Graph Refactorings(图形重构）

节点合并、属性规范化与分类等

#### 4.7 Virtual Nodes/Rels(虚拟节点/关系）

提供虚拟图的创建

#### 4.8 Cypher Operations(Cypher操作）

单个和多个的Cypher语句运行和脚本运行

#### 4.9 Triggers(触发器）

与关系型数据库的触发器的理解方式一样， 即当有数据改变时，下面进行的一系列动作就叫做触发器。



## 三、APOC数据集成-JDBC

### 1. 什么是JDBC


apoc.load.jdbc:可以访问提供JDBC驱动程序的数据库，并执行查询。其将结果变成以一行数据为单位的数据流。然后可以使用这些行来更新或创建图形数据结构

2. APOC JDBC 语法

```sql
call
apoc.load.jdbc("jdbc:mysql://(IP):(PORT)/(DBNAME)?user={USERNAME)&password={PASSWORD}","(TABLENAME)") yield row
create (b:Black{number:row.black_id, type:row.type})
```

创建节点的语句，使用row.调用每一行中的具体字段

Black实体中有两个属性，一个是number，一个是type。

实体根据颜色可以得到它是否是新创建的，灰色的是新创建的，而红色的是之前导入的。可以根据自己需要进行修改颜色。

![image-20210112185318866](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112185318.png)

![image-20210112185327525](https://gitee.com/zgf1366/pic_store/raw/master/img/20210112185327.png)

