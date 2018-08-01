# MongoDB集群搭建教程



​		    本文内容参考自：[MongodDB 3.4 集群搭建：分片+副本集](https://www.cnblogs.com/ityouknow/p/7344005.html)

​						   [MongoDB集群搭建与介绍](https://blog.csdn.net/wangshuang1631/article/details/53857319)

​						   [The MongoDB 4.0 Manual](The MongoDB 4.0 Manual)

​						   [Nosql简介 Redis，Memchche,MongoDb的区别](https://www.cnblogs.com/lina520/p/7919551.html)

### NoSQL与MongoDB:

------

**MongoDB**是一个基于分布式文件存储的数据库。介于关系型数据库和非关系型数据库之间。

**NoSQL**（Not only SQL）指的是非关系型数据库。

具体的科普链接传送门：[Nosql简介 Redis，Memchche,MongoDb的区别](https://www.cnblogs.com/lina520/p/7919551.html)



### MongoDB分片+副本集的意义：

------

请参考[Mongodb集群搭建与介绍](https://blog.csdn.net/wangshuang1631/article/details/53857319) 这篇文章的说明，了解什么是分片以及什么是副本集。

了解分片以及副本集的概念之后，不难理解分片+副本集的意义：

> * 通过分片加副本集的设置分片存储数据，提高数据处理效率
> * 通过在不同的服务器间创建副本，降低数据丢失的风险
> * 在主服务器不能正常工作的时候自动切换到副服务器继续提供服务

![副本集示意图1](https://ws1.sinaimg.cn/large/006tNc79ly1ftubewekxnj30ec09umy6.jpg)

![副本集示意图2](https://ws1.sinaimg.cn/large/006tNc79ly1ftubi9pehjj30e809zdgq.jpg)



​						     				    副本集的作用



![分片示意图](https://ws3.sinaimg.cn/large/006tNc79ly1ftubkhcumbj30aw09y3z3.jpg)



​						    				    分片的作用



### MongoDB分片+副本集搭建

------

![img](http://www.ityouknow.com/assets/images/2017/bigdata/sharded-cluster-production-architecture.bakedsvg.svg)

从图中可以看到有四个组件：mongos、config server、shard、replica set。

**mongos**，数据库集群请求的入口，所有的请求都通过mongos进行协调，不需要在应用程序添加一个路由选择器，mongos自己就是一个请求分发中心，它负责把对应的数据请求请求转发到对应的shard服务器上。在生产环境通常有多mongos作为请求的入口，防止其中一个挂掉所有的mongodb请求都没有办法操作。

**config server**，顾名思义为配置服务器，存储所有数据库元信息（路由、分片）的配置。mongos本身没有物理存储分片服务器和数据路由信息，只是缓存在内存里，配置服务器则实际存储这些数据。mongos第一次启动或者关掉重启就会从 config server 加载配置信息，以后如果配置服务器信息变化会通知到所有的 mongos 更新自己的状态，这样 mongos 就能继续准确路由。在生产环境通常有多个 config server 配置服务器，因为它存储了分片路由的元数据，防止数据丢失！

**shard**，分片（sharding）是指将数据库拆分，将其分散在不同的机器上的过程。将数据分散到不同的机器上，不需要功能强大的服务器就可以存储更多的数据和处理更大的负载。基本思想就是将集合切成小块，这些块分散到若干片里，每个片只负责总数据的一部分，最后通过一个均衡器来对各个分片进行均衡（数据迁移）。

**replica set**，中文翻译副本集，其实就是shard的备份，防止shard挂掉之后数据丢失。复制提供了数据的冗余备份，并在多个服务器上存储数据副本，提高了数据的可用性， 并可以保证数据的安全性。

**Arbiter**，中文翻译仲裁节点，是复制集中的一个MongoDB实例，它并不保存数据。仲裁节点使用最小的资源并且不要求硬件设备，不能将Arbiter部署在同一个数据集节点中，可以部署在其他应用服务器或者监视服务器中，也可部署在单独的虚拟机中。为了确保复制集中有奇数的投票成员（包括primary），需要添加仲裁节点做为投票，否则primary不能运行时不会自动切换primary。

简单了解之后，我们可以这样总结一下，应用请求mongos来操作mongodb的增删改查，配置服务器存储数据库元信息，并且和mongos做同步，数据最终存入在shard（分片）上，为了防止数据丢失同步在副本集中存储了一份，仲裁在数据存储到分片的时候决定存储到哪个节点。



**环境准备**

------

- 操作系统及MongoDB安装包（本文以ubuntu环境下的MongoDB-3.4.6为例）：

```shell
wget http://downloads.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.4.6.tgz
```

- 服务器IP及端口分配

| 服务器（内） | config | shard1 | shard2 | shard3 | mongos |
| :----------: | :----: | :----: | :----: | :----: | :----: |
| 192.188.62.4 | 21000  | 27001  | 27002  | 27003  | 20000  |
| 192.188.62.5 | 21000  | 27001  | 27002  | 27003  | 20000  |
| 192.188.62.6 | 21000  | 27001  | 27002  | 27003  | 20000  |

- 服务器规划

|  服务器192.188.62.4  |  服务器192.188.62.5  |  服务器192.188.62.6  |
| :------------------: | :------------------: | :------------------: |
|        mongos        |        mongos        |        mongos        |
|    config server     |    config server     |    config server     |
| shard server1 主节点 | shard server1 副节点 |  shard server1 仲裁  |
|  shard server2 仲裁  | shard server2 主节点 | shard server2 副节点 |
| shard server3 副节点 |  shard server3 仲裁  | shard server3 主节点 |

- 相关配置文件及脚本下载（除了mongos.config以外，其他配置文件都不需要更改）

```shell
git clone https://github.com/Yida147/mongo.git
```

**集群搭建**

------

[完整的无省略版的集群配置文件书写教程](https://www.cnblogs.com/ityouknow/p/7344005.html)

***1.*** ***安装并解压mongodb***

```shell
tar -xzvf mongodb-linux-x86_64-3.4.6.tgz -C /usr/local/
cd /usr/local
# 重命名
mv mongodb-linux-x86_64-3.4.6 mongodb
mv mongo/shell /usr/local/mongodb/
# 更改权限
sudo chmod -R 777 mongodb
```

分别在每台机器建立conf、mongos、config、shard1、shard2、shard3六个目录，因为mongos不存储数据，只需要建立日志文件目录即可。这里我们利用脚本创建目录即可。

```shell
cd /usr/local/mongodb/shell
./mkdir.sh
```

配置环境变量。

```shell
vim /etc/profile
# 内容
export MONGODB_HOME=/usr/local/mongodb
export PATH=$MONGODB_HOME/bin:$PATH
# 使立即生效
source /etc/profile
```

***2.*** ***移动配置文件到对应的配置文件夹***

```shell
mv mongo/conf /usr/local/mongodb/
```

***3.*** ***通过脚本启动三台服务器的config server、shard1、shard2、shard3、mongos***

```shell
cd /usr/local/mongodb/shell
./startmongoserver.sh
```

***4.*** ***登录任意一台配置服务器，初始化configs副本集***

```shell
#连接
mongo --port 21000
#config变量
config = {
...    _id : "configs",
...     members : [
...         {_id : 0, host : "192.188.62.4:21000" },
...         {_id : 1, host : "192.188.62.5:21000" },
...         {_id : 2, host : "192.188.62.6:21000" }
...     ]
... }

#初始化副本集
rs.initiate(config)
```

其中，"_id" : "configs"应与配置文件中配置的 replicaction.replSetName 一致，"members" 中的 "host" 为三个节点的 ip 和 port

 ***5.*** ***登录任意一台配置服务器，初始化shard1副本集***

```shell
mongo --port 27001
#使用admin数据库
use admin
#定义副本集配置，第三个节点的 "arbiterOnly":true 代表其为仲裁节点。
config = {
...    _id : "shard1",
...     members : [
...         {_id : 0, host : "192.188.0.4:27001" },
...         {_id : 1, host : "192.188.0.5:27001" },
...         {_id : 2, host : "192.188.0.6:27001" , arbiterOnly: true }
...     ]
... }
#初始化副本集配置
rs.initiate(config)
```

***6.*** ***登录任意一台配置服务器，初始化shard2副本集***

```shell
mongo --port 27002
#使用admin数据库
use admin
#定义副本集配置
config = {
...    _id : "shard2",
...     members : [
...         {_id : 0, host : "192.188.62.4:27002"  , arbiterOnly: true },
...         {_id : 1, host : "192.188.62.5:27002" },
...         {_id : 2, host : "192.188.62.6:27002" }
...     ]
... }

#初始化副本集配置
rs.initiate(config);
```

***7.*** ***登录任意一台配置服务器，初始化shard3副本集***

```shell
mongo --port 27003
#使用admin数据库
use admin
#定义副本集配置
config = {
...    _id : "shard3",
...     members : [
...         {_id : 0, host : "192.188.62.4:27003" },
...         {_id : 1, host : "192.188.62.5:27003" , arbiterOnly: true},
...         {_id : 2, host : "192.188.62.6:27003" }
...     ]
... }

#初始化副本集配置
rs.initiate(config);
```

***8.*** ***启用分片***

```shell
mongo --port 20000
#使用admin数据库
user  admin
#串联路由服务器与分配副本集
sh.addShard("shard1/192.188.62.4:27001,192.188.62.5:27001,192.188.62.6:27001")
sh.addShard("shard2/192.188.62.4:27002,192.188.62.5:27002,192.188.62.6:27002")
sh.addShard("shard3/192.188.62.4:27003,192.188.62.5:27003,192.188.62.6:27003")
#查看集群状态
sh.status()
```

***9.*** ***测试***

目前配置服务、路由服务、分片服务、副本集服务都已经串联起来了，但我们的目的是希望插入数据，数据能够自动分片。连接在mongos上，准备让指定的数据库、指定的集合分片生效。

```shell
#指定testdb分片生效
db.runCommand( { enablesharding :"testdb"});
#指定数据库里需要分片的集合和片键
db.runCommand( { shardcollection : "testdb.table1",key : {id:"hashed"} } )
```

我们设置testdb的 table1 表需要分片，根据 id 自动分片到 shard1 ，shard2，shard3 上面去。要这样设置是因为不是所有mongodb 的数据库和表 都需要分片！

测试分片配置结果

```shell
mongo  127.0.0.1:20000
#使用testdb
use  testdb;
#插入测试数据
for (var i = 1; i <= 100000; i++)
db.table1.save({id:i,"test1":"testval1"});
#查看分片情况
db.table1.stats()
```