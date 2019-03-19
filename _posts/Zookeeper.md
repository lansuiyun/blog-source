---
title: Zookeeper
date: 2019-03-14 11:10:03
tags: [Zookeeper]
categories: 分布式
---
### 什么是zookeeper
> ZooKeeper is a distributed, open-source coordination service for distributed applications. It exposes a simple set of primitives that distributed applications can build upon to implement higher level services for synchronization, configuration maintenance, and groups and naming.

ZooKeeper 是一个开源的分布式协调服务。它实现了一组原语集，在此基础上，分布式应用可以构建更高级别的服务，如同步、配置维护、命名服务等。
<!-- more-->
ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。Zookeeper 一个最常用的使用场景就是用于担任服务生产者和服务消费者的注册中心。 服务生产者将自己提供的服务注册到Zookeeper中心，服务的消费者在进行服务调用的时候先到Zookeeper中查找服务，获取到服务生产者的详细信息之后，再去调用服务生产者的内容与数据。
Zookeeper本身是高可用的，集群中超过半数的服务存活即可提供服务。
![](https://zookeeper.apache.org/doc/current/images/zkservice.jpg)
ZooKeeper 是有序的。所有事物操作都有一个反应顺序的数字编号。
Zookeeper时快速的。尤其是在以读为主的系统中。
### 数据模型
Zookeeper使用了一种类似与文件系统的命名空间。每个节点称为`znode`。
![](https://zookeeper.apache.org/doc/current/images/zknamespace.jpg)

#### Zookeeper中的时间
ZooKeeper记录时间有多种方式：
1. Zxid
Zookeeper中每个改变stat都会收到一个zxid（ZooKeeper Transaction Id）格式的时间戳。这个时间戳时全局有序的。如果zxid1小于zxid2，说明zxid1发生早于zxid2.
2. Version number
节点每次改变时，版本号中的一个就会自动增长。有三种版本号：version 节点数据版本号、cversion 子节点版本号、aversion ACL版本号。
3. Ticks
Zookeeper使用ticks定义server间如状态上传、回话过期、连接超时等事件的时间。
4. Real time
Zookeeper会把节点创建、修改的时间戳放入到stat结构中，除此之外，不使用真实事件。
#### 节点结构
与文件系统不同的是，Zookeeper中每个节点都有与自己关联的数据。节点被设计来存储协调数据：状态信息、配置，位置信息等，这些信息通常比较小，最大为1M。每个节点包含以下数据：
1. stat
包含节点的版本信息，ACL信息，时间戳等，具体如下：
  - czxid 节点创建的zxid
  - mzxid 节点最后被修改的zxid
  - pzxid 子节点最后被修改的zxid
  - ctime 从新纪元开始到节点被创建的时间，单位为毫秒
  - mtime 从新纪元开始到节点被修改的时间，单位为毫秒
  - version 节点数据版本号
  - cversion 子节点版本号
  - aversion 节点ACL版本号
  - ephemeralOwner 当节点为临时节点时，为节点所有者会话id，否则为0.
  - dataLength 节点数据长度
  - numChildren 子节点个数
```
cZxid = 0x300000002
ctime = Wed Mar 13 10:29:45 CST 2019
mZxid = 0x300000004
mtime = Wed Mar 13 10:30:46 CST 2019
pZxid = 0x30000000b
cversion = 5
dataVersion = 2
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 1
```
2. data
与节点关联的数据。节点数据的读、写都具有原子性。读获取节点关联的所有数据，写替换所有数据。每个节点都有Access Control List (ACL)来控制谁可以做什么。
3. children 子节点

#### 节点类型
按照时效分类有两种节点：
1. 永久节点。创建后除非删除，否则节点一直存在。
2. 临时节点。该类节点的生命周期依赖于创建它的session。session过期后，临时节点就会被删除。临时节点不能有子节点。

按照顺序分类也有两种节点：
1. 正常节点。
2. 顺序节点。Zookeeper在创建顺序节点时会在节点路径后拼接一个递增的计数。对应父节点来说，这个计数时唯一的。计数格式为`%010d`，10位数字，没有则补0，例如：<path>0000000001。计数最大为`2147483647`，超出后将溢出。

这样，Zookeeper就一共有4种节点。

### Watches
客户端可以在节点上注册`watches`，当节点有变化时会触发watch，通知客户端，并清空watch，即watch是一次性的。
Zookeeper中所有的读取操作`getData()`、`getChildren()`、`exists()`都可以通过设置是否watch。关于watch有三个关键点需要考虑：
1. watch只会触发一次。当客户端收到watch事件后，触发重新设置一个新的watch，否则不会收到任何watch事件。
2. watch事件是异步发送给客户端的。Zookeeper提供一个关于顺序的保证：如果一个客户端已经注册了watch，在收到watch事件前不会看到任何的改变。
3. `getData()`、`exists()` 设置的时数据的wathc，`getChildren()`设置的时子节点的wathc。

watches是有Zookeeper server在本地管理。当客户端与服务器断开连接后，不会收到任何的watch事件。当客户端重连后，之后注册的watch将会被重新注册、触发。有一种情况会导致wathc被丢失：节点是否存在的watch在客户端离线期间被创建、删除。

#### watch触发的事件与操作
1. 创建事件 `exists`
2. 删除事件 `exists`、`getData`、`getChildren`
3. 改变事件 `exists`、`getData`
4. 子节点事件 `getChildren`


### ACL
ZooKeeper通过ACL控制节点的访问，ACL提供一组ids。ACL从属于特定的节点，并且没有递归性。例如，`/app`只能被ip:172.16.16.1读取，但`/app/status`可以被所有客户端读。
ZooKeeper支持可插拔的身份验证方案。Ids使用`scheme:id`的格式,`scheme`是验证方案。例如`ip:172.16.16.1`。ACL由成对的`scheme:expression, perms`组成，例如:`ip:19.22.0.0/16, READ`。

#### ACL权限
- CREATE 创建子节点
- READ 可以获取节点数据，子节点列表
- WRITE 写入节点数据
- DELETE 删除子节点
- ADMIN 设置节点权限

#### ACL内置方案
- world 所有人
- auth 已经通过人证的用户
- digest username:password
- ip 客户端ip，格式为`addr/bits`

#### 设置acl
1. world
setAcl <path> world:anyone:<acl>
`setAcl /acl world:anyone:cdrwa`
2. auth
addauth digest <user>:<password> #添加认证用户
setAcl <path> auth:<user>:<acl>
```
addauth digest root:123456
setAcl /acl auth:root:cdrwa
```
3. digest
setAcl <path> digest:<user>:<password>:<acl>
其中`pass`需要对`用户名:密码`进行SHA1、BASE64运算后得出，可以通过指令：
`echo -n user:password | openssl dgst -binary -sha1 | openssl base64`
```
setAcl /acl digest:root:dqFD4/T6OaSbifsDlGaqTwLweYE=:cdrwa
```
4. ip
setAcl <path> ip:<ip>:<acl>    
`setAcl /acl ip:192.168.1.1:cdrwa`

#### java客户端相关
##### 创建带有ACL的节点
ZooDefs.Ids中提供了几种ACL:
`OPEN_ACL_UNSAFE` 对所有人开放
`CREATOR_ALL_ACL` 节点创建者有全部权限
`READ_ACL_UNSAFE` 所有人可读
```
zooKeeper.create("/acl", "acl".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
```
自定义：
```
List<ACL> acls = new ArrayList<ACL>();  // 权限列表
// 第一个参数是权限scheme，第二个参数是加密后的用户名和密码
Id root = new Id("digest", DigestAuthenticationProvider.generateDigest("root:123"));
Id user = new Id("digest", DigestAuthenticationProvider.generateDigest("user:456"));
// 所有权限
acls.add(new ACL(ZooDefs.Perms.ALL, user1));  
// 读权限
acls.add(new ACL(ZooDefs.Perms.READ, user2));  
```

权限验证：
```
zooKeeper.addAuthInfo("digest", "root:123".getBytes());
```

### 一致性保证
ZooKeeper时一个高性能、可伸缩的服务。读写都非常快，但读比写快，因为读可以使用旧数据。其提供以下一致性保证：
1. 顺序一致性
客户端更新的顺序与发送的顺序一致
2. 原子性
更新不是成功就是失败，没有部分结果
3. 单一系统镜像
不论客户端连接到集群中的哪一个服务器，都能看到同样的视图
4. 可靠性
一旦更新被执行，其结果将会被持久化，直到有客户端重写了更新。这个保证有两个结论：
  - 如果一个客户端收到成功更新的返回，说明更新已经被执行；
  - Any updates that are seen by the client, through a read request or successful update, will never be rolled back when recovering from server failures.
5. 及时性
客户端看到的系统视图在一定时间（大约几十秒）会更新。在这个时间边界内，客户端要么看到更新，要么失去连接。

### 指令
```
stat path [watch]
set path data [version]
ls path [watch]
delquota [-n|-b] path
ls2 path [watch]
setAcl path acl
setquota -n|-b val path
history
redo cmdno
printwatches on|off
delete path [version]
sync path
listquota path
rmr path
get path [watch]
create [-s] [-e] path data acl
addauth scheme auth
quit
getAcl path
close
connect host:port
```


### 配置参数
- `tickTime` zookeeper中的基础时间单元，单位为毫秒。
- `dataDir` 存储内存数据快照的位置。除非另外指定，则数据更新的事物日志也存放在这里。
- `clientPort` 客户端连接端口。
- `initLimit` follower连接到leader初始化的超时时间，以为`tickTime`为单位。
- `syncLimit` leader与follower之间请求、应答超时时间，以为`tickTime`为单位。
- `server.A=B:C:D`
  - `A` 服务器编号，与`dataDir`下`myid`文件中的编号保持一致。
  - `B` ip
  - `C` 服务器之间通信端口
  - `D` Leader选举端口

#### 其他配置
- `myid`
集群模式下处理`zoo.cnf`还需要在`dataDir`下定义`myid`文件，在`myid`中会写入server编号。这样，server在启动时会搜索`myid`文件，从而确定自己属于哪个server。

### zookeeper搭建
zookeeper有三种模式
- 单机模式 只运行一个实例
- 伪集群模式 在一台设备上运行多个实例
- 集群模式 运行于集群上，所有实例公用统一的配置文件。

#### 伪集群
伪集群即在一台服务器上模拟集群运行。
在一台服务器上部署3个zookeeper server。每台server需要设置不同的`dataDir`，`dataLogDir`、`clientPort`。
##### 配置
在zookeeper/conf下新建三个conf，分别为`zoo1.cnf`,`zoo2.cnf`,`zoo3.cnf`.`zoo1.cnf`如下：
```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/fei/dbFile/zookeeper/1
dataLogDir=/home/fei/temp/logs/zookeeper/1
clientPort=2187

server.1=localhost:2287:3387
server.2=localhost:2288:3388
server.3=localhost:2289:3389
```
在conf1的data目录下创建`myid`文件，并写入自身编号，即`server.x`对应的`x`。
依次创建`zoo2.cnf`,`zoo3.cnf`，并修改相关配置。

##### 启动
```
zkServer.sh start zoo1.cnf
zkServer.sh start zoo2.cnf
zkServer.sh start zoo3.cnf
```

### 集群
与伪集群类似，但可以使用相同的配置文件

### 其他
#### 集群内服务器数量
ZooKeeper通过复制来实现高可用性，只要集合体中半数以上的机器处于可用状态，它就能够提供服务。所以集群内最好使用>=3的奇数台服务器。例如3台、4台server情况下，都最多允许1台server挂掉，可靠性一致。

#### 集群角色
zookeeper中集群分为`Leader`、`Follower`、`Observer`三种角色
![](https://user-gold-cdn.xitu.io/2018/9/11/165c68aae5befdae?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
集群中所有的写操作都有Leader执行，Follower、Observer可以提供读操作。Follower参与Leader的选举过程，写操作的`写过半成功`过程，而Observer不参与。引入Observer是为了提供集群的读性能。

#### Zookeeper原理
Zookeeper核心时原子广播机制，这个机制保证了各个server之间的同步。实现这个机制的协议叫做Zab协议，其核心内容为：
所有的事务请求必须由Leader来协调处理。Leader服务器负责将一个客户端请求转化为事务提议（Proposal），并将该proposal分发给集群所有的follower服务器。之后Leader服务器需要等待所有的follower服务器的反馈，一旦超过了半数的follower服务器进行了正确反馈后，那么Leader服务器就会再次向所有的follower服务器分发commit消息，要求其将前一个proposal进行提交。

Zab协议有两种模式，它们分别是恢复模式和广播模式。

1. 恢复模式
当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和server具有相同的系统状态。

2. 广播模式
一旦Leader已经和多数的Follower进行了状态同步后，他就可以开始广播消息了，即进入广播状态。这时候当一个Server加入ZooKeeper服务中，它会在恢复模式下启动，发现Leader，并和Leader进行状态同步。待到同步结束，它也参与消息广播。ZooKeeper服务一直维持在Broadcast状态，直到Leader崩溃了或者Leader失去了大部分的Followers支持。
Broadcast模式极其类似于分布式事务中的2pc（two-phrase commit 两阶段提交）：即Leader提起一个决议，由Followers进行投票，Leader对投票结果进行计算决定是否通过该决议，如果通过执行该决议（事务），否则什么也不做。
在广播模式ZooKeeper Server会接受Client请求，所有的写请求都被转发给领导者，再由领导者将更新广播给跟随者。当半数以上的跟随者已经将修改持久化之后，领导者才会提交这个更新，然后客户端才会收到一个更新成功的响应。这个用来达成共识的协议被设计成具有原子性，因此每个修改要么成功要么失败。
