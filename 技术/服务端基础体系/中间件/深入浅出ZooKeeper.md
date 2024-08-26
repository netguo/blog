## 一、ZooKeeper起源和应用
### 1.1 ZooKeeper起源
ZooKeeper 最早起源于雅虎研究院的一个研究小组。当时，雅虎内部很多大型系统基本都需要依赖一个类似的系统来进行分布式协调，但是这些系统往往都存在分布式单点问题。所以，雅虎的开发人员就试图开发一个通用的无单点问题的分布式协调框架，以便让开发人员将精力集中在处理业务逻辑上。
### 1.2 ZooKeeper应用
Zookeeper的常见的应用场景

- 配置中心：通常在分布式系统或集群中，所有节点的配置应该一致，比如Hadoop集群，要求对配置的修改，能够快速同步到各个节点中，可以通过 Zookeeper 实现
- 集群管理：一个集群各个节点信息，注册到Zookeeper，集群节点可以观察到其它集群节点的状态，在集群选举，主从协调等场景使用。
- 服务注册中心：服务提供者将自己的服务信息（例如IP地址、端口号等）注册到ZooKeeper中，而服务消费者则通过查询ZooKeeper来发现可用的服务。例如dubbo。
- 分布式锁：在一个永久节点下创建有序的临时子节点后，根据编号顺序，最小顺序的子节点获取到锁，其他子节点由小到大监听前一个节点。
### 1.3 ZooKeeper特点

- 小数据存储 （小而重要）
- 快速的读写
- 可靠watch机制
- 高可用性：ZK采用了多副本机制，数据在多个节点上进行复制存储
- 强一致性：所有客户端对共享数据的读取都获取最新的一致的视图。每次的写入操作都会经过 Leader节点，确保数据在所有节点同步
- 顺序性： ZK保证了所有操作请求的顺序性
## 二、ZooKeeper的设计
### 2.1 Zookeeper的数据结构
ZooKeeper 是一个树形目录服务,其数据模型和Unix的文件系统目录树很类似，拥有一个层次化结构。Zookeeper这里面的每一个节点都被称为： ZNode，每个节点上都会保存自己的数据和节点信息。节点可以拥有子节点，同时也允许少量（1MB）数据存储在该节点之下。
节点可以分为四大类：

- PERSISTENT 持久化节点
- EPHEMERAL 临时节点
- PERSISTENT_SEQUENTIAL 持久化顺序节点 
- EPHEMERAL_SEQUENTIAL 临时顺序节点

临时：回话结束后删除节点
顺序：根据先后顺序，会在节点之后带上一个数值，越后执行数值越大，适用于分布式锁的应用场景- 单调递增
![highlight.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1709002642284-f101a3a0-962f-4d20-a0c7-cb1d9088647a.png#averageHue=%23c6c6c6&clientId=ua1a7eb46-3c24-4&from=ui&id=u09786971&originHeight=618&originWidth=1258&originalType=binary&ratio=2&rotation=0&showTitle=false&size=179380&status=done&style=none&taskId=u5f2dd731-846e-4214-bbe5-3a19f0b9fcc&title=)
### 2.3 节点操作
支持节点的创建/查看/删除/修改节点
### 2.4 Watch机制
#### 2.4.1 Watch机制的使用
ZooKeeper 中引入了Watcher机制来实现了发布/订阅功能能，能够让多个订阅者同时监听某一个对象，当一个对象自身状态变化时，会通知所有订阅者。可以监听事件：

- None
- NodeCreated： 节点创建
- NodeDeleted： 节点删除
- NodeDataChanged：节点数据变化
- NodeChildrenChanged：子节点列表变化
- DataWatchRemoved：节点监听被移除
- ChildWatchRemoved：子节点监听被移除
- PersistentWatchRemoved

ZooKeeper提供了三种Watch方法：getData、exist 、getChildren。
#### 2.4.2 Watch机制原理
ZooKeeper 中 Watch 机制的，大体上ZooKeeper 实现的方式是通过客服端和服务端分别创建有观察者的信息列表。客户端调用 getData、exist 、getChildren接口时。
1）首先将对应的 Watch 事件放到本地的 ZKWatchManager 中进行管理。服务端在接收到客户端的请求后根据请求类型判断是否含有 Watch 事件，并将对应事件放到 WatchManager 中进行管理。
2）在事件触发的时候服务端通过节点的路径信息查询相应的 Watch 事件通知给客户端，客户端在接收到通知后，首先查询本地的 ZKWatchManager 获得对应的 Watch 信息处理回调操作。
这种设计不但实现了一个分布式环境下的观察者模式，而且通过将客户端和服务端各自处理 Watch 事件所需要的额外信息分别保存在两端，减少彼此通信的内容，提升了服务的处理性能。
客户端的Watch是一次性的，监听完成一个事件后会被删除，需要重新注册。Curator引入了Cache机制，实现长期的监听。
### 2.5 权限控制
为了保证zookeeper中的数据的安全性，避免误操作带来的影响。Zookeeper提供了一套ACL权限控制机制来保证数据的安全。ACL机制：scheme:id:perm采来标识。Scheme（权限模式），标识授权策略；ID（授权对象）；Permission：授予的权限。
2.5.1 Scheme 权限模式

- **world:** 默认方式，相当于全部都能访问。
- **auth**：代表已经认证通过的用户。
- **digest**：即用户名:密码这种方式认证，这也是业务系统中最常用的。
- **ip**：通过ip地址来做权限控制
#### 2.5.2 ID授权对象
指的是具体授权的对象，例如ip，用户名:密码
#### 2.5.3 Permission

- Create 允许对子节点Create 操作
- Read 允许对本节点GetChildren 和GetData 操作
- Write 允许对本节点SetData 操作
- Delete 允许对子节点Delete 操作
- Admin 允许对本节点setAcl 操作
### 2.6 Java客户端
原生客户端Zookeeper Client和Curator，日常项目使用建议采用Curator。
Curator是Netflix公司开源的一套zookeeper客户端框架，Curator是对Zookeeper支持最好的客户端框架。Curator封装了大部分Zookeeper的功能，比如Leader选举、分布式锁等，减少了技术人员在使用Zookeeper时的底层细节开发工作。
Curator框架主要解决了三类问题：

- 封装ZooKeeper Client与ZooKeeper Server之间的连接处理（提供连接重试机制等）。
- 提供了一套Fluent风格的API，并且在Java客户端原生API的基础上进行了增强（创捷多层节点、删除多层节点等）。
- 提供ZooKeeper各种应用场景（分布式锁、leader选举、共享计数器、分布式队列等）的抽象封装。
### 2.7 Zookeeper集群
zookeeper集群中的节点有三种角色

- Leader：处理集群的所有事务请求（增删改），集群中只有一个Leader。
- Follower：只能处理读请求，参与Leader选举。
- Observer：只能处理读请求，提升集群读的性能，但不能参与Leader选举。

写请求只能有Leader执行，做数据同步。如果写请求发送到Follower，会转发到Leader，由Leader执行写操作。
## 三、Zab概述
全称是Zookeeper Atomic Broadcast(Zookeeper原子广播)。Zookeeper 是通过Zab算法来保证分布式事务的最终一致性。
### 3.1 起源-Paxos
Paxos算法由拜占庭将军问题作者Leslie Lamport于1998年提出，2013年Leslie Lamport获得图灵奖。Paxos公认为分布式一致性算法中最复杂和难理解的算法之一。
Zab基于Paxos算法基础上进行优化和简化，Paxos算法此处不再赘述。Zab延续了Paxos核心思想，少数服从多数，只要超过一半的机器认可某一消息，最终所有机器都会接受这消息。少数服从多数，解决了“脑裂”问题，此处不再概述。
### 3.2 Zab协议流程
ZooKeeper通过Zab协议实现了分布式一致性和顺序性。
#### 3.2.1 基本概念
集群角色：Leader、Follower、Observer
成员状态：Looking、Following、Leading、Observing
运行阶段：选举阶段、选举发现、数据同步、消息广播
事物标识符：Epoch(任期编号)、Countor(事务ID，zxid)
#### 3.2.2 运行阶段流程
1）成员选举，每一个Follower给自己投票，然后广播投票信息。投票原则，先比较epoch，投票大的；再比较zxid，投票大的；最后比较myid投票大的。最终获得半数以上的投票的Follower成为新的Leader。因为每一个节点的myid不同，所以最后一定会有一个Follower成为新的Leader。
2）成员发现，所有Follower向Leader同步epoch信息，获得所有Follower的epoch信息后，Leader更新自己epoch，并给Follower发送确认信息，Follower发送自己的epoch，zxid信息。此时Leader知晓了所有Follower的epoch，zxid信息。
3）数据同步，新的Leader向Follower同步信息。同步信息分为：

- Diff：相同的epoch，同步差异数据
- Snap：差异数据比较大，通过快照形式全量同步
- Trunc：回滚同步，旧的Leader宕机一段时间恢复，部分数据Follower未commit，此时需要回滚。
- Trunc+Diff：回滚并同步差异。

 Follower跟新的Leader数据同步完成后，新Leader更新任期epoch，zxid设置为0，并通知所有Follower更新，并确认更新。
4）此时新的任期，新的消息广播阶段。Leader接受到写/数据变更请求，为消息生成zxid，数据先写入磁盘(写前日志)，并通知Follower变更，Follower收到消息后，返回ack。Leader收到半数以上Follower的ack信息后，发送Commit请求。并从磁盘读取数据，变更内存中Znode数据。
此时考虑两种情况：如果Leader发送Commit信息后，所有Follower未收到或者部分Follower收到信息后，此时Leader宕机，最终集群数据会是什么样子，作为思考题，本文不给出答案。
#### ![zab.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1709110345353-6519d335-837a-4cd1-a72f-417d5868a902.png#averageHue=%23f8f7f7&clientId=u08052d84-5004-4&from=ui&id=M20Ek&originHeight=1231&originWidth=792&originalType=binary&ratio=2&rotation=0&showTitle=false&size=147296&status=done&style=none&taskId=u6d848e50-5004-4ff1-85ae-890570e6341&title=)
