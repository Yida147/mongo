# MongoDB集群搭建教程



​		    本文内容参考自：[MongodDB 3.4 集群搭建：分片+副本集](https://www.cnblogs.com/ityouknow/p/7344005.html)

​						   [MongoDB集群搭建与介绍](https://blog.csdn.net/wangshuang1631/article/details/53857319)

​						   [The MongoDB 4.0 Manual](The MongoDB 4.0 Manual)

​						   [Nosql简介 Redis，Memchche,MongoDb的区别](https://www.cnblogs.com/lina520/p/7919551.html)

### NoSQL与MongoDB:

------

​	MongoDB是一个基于分布式文件存储的数据库。介于关系型数据库和非关系型数据库之间。

​	NoSQL（Not only SQL）指的是非关系型数据库。

​	具体的科普链接传送门：[Nosql简介 Redis，Memchche,MongoDb的区别](https://www.cnblogs.com/lina520/p/7919551.html)



### MongoDB分片+副本集的意义：

------

​	请参考[Mongodb集群搭建与介绍](https://blog.csdn.net/wangshuang1631/article/details/53857319) 这篇文章的说明，了解什么是分片以及什么是副本集。

​	了解分片以及副本集的概念之后，不难理解分片+副本集的意义：

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

**环境准备**

------

- 操作系统及MongoDB安装包（本文以ubuntu环境下的MongoDB-3.4.6为例）：

```shell
$ wget http://downloads.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.4.6.tgz
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

```

```

**集群搭建**

------

***1.*** ***安装mongodb***



