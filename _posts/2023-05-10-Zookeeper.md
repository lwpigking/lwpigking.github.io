---
layout: post
title: Zookeeper入门
categories: [Zookeeper]
description: Zookeeper很重要！
keywords: Zookeeper
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# Zookeeper

## 简介及核心概念

### 简介

Zookeeper 是一个开源的分布式协调服务，目前由 Apache 进行维护。Zookeeper 可以用于实现分布式系统中常见的发布/订阅、负载均衡、命令服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。它具有以下特性：

+ **顺序一致性**：从一个客户端发起的事务请求，最终都会严格按照其发起顺序被应用到 Zookeeper 中；
+ **原子性**：所有事务请求的处理结果在整个集群中所有机器上都是一致的；不存在部分机器应用了该事务，而另一部分没有应用的情况；
+ **单一视图**：所有客户端看到的服务端数据模型都是一致的；
+ **可靠性**：一旦服务端成功应用了一个事务，则其引起的改变会一直保留，直到被另外一个事务所更改；
+ **实时性**：一旦一个事务被成功应用后，Zookeeper 可以保证客户端立即可以读取到这个事务变更后的最新状态的数据。



### 设计目标

Zookeeper 致力于为那些高吞吐的大型分布式系统提供一个高性能、高可用、且具有严格顺序访问控制能力的分布式协调服务。它具有以下四个目标：

#### 2.1 目标一：简单的数据模型

Zookeeper 通过树形结构来存储数据，它由一系列被称为 ZNode 的数据节点组成，类似于常见的文件系统。不过和常见的文件系统不同，Zookeeper 将数据全量存储在内存中，以此来实现高吞吐，减少访问延迟。

![zookeeper-zknamespace](/picture/pictures/zookeeper-zknamespace.jpg)

#### 2.2 目标二：构建集群

可以由一组 Zookeeper 服务构成 Zookeeper 集群，集群中每台机器都会单独在内存中维护自身的状态，并且每台机器之间都保持着通讯，只要集群中有半数机器能够正常工作，那么整个集群就可以正常提供服务。

![zookeeper-zkservice](/picture/pictures/zookeeper-zkservice.jpg)

#### 2.3 目标三：顺序访问

对于来自客户端的每个更新请求，Zookeeper 都会分配一个全局唯一的递增 ID，这个 ID 反映了所有事务请求的先后顺序。

#### 2.4 目标四：高性能高可用

ZooKeeper 将数据存全量储在内存中以保持高性能，并通过服务集群来实现高可用，由于 Zookeeper 的所有更新和删除都是基于事务的，所以其在读多写少的应用场景中有着很高的性能表现。



### 核心概念

#### 3.1 集群角色

Zookeeper 集群中的机器分为以下三种角色：

+ **Leader** ：为客户端提供读写服务，并维护集群状态，它是由集群选举所产生的；
+ **Follower** ：为客户端提供读写服务，并定期向 Leader 汇报自己的节点状态。同时也参与写操作“过半写成功”的策略和 Leader 的选举；
+ **Observer** ：为客户端提供读写服务，并定期向 Leader 汇报自己的节点状态，但不参与写操作“过半写成功”的策略和 Leader 的选举，因此 Observer 可以在不影响写性能的情况下提升集群的读性能。

#### 3.2 会话

Zookeeper 客户端通过 TCP 长连接连接到服务集群，会话 (Session) 从第一次连接开始就已经建立，之后通过心跳检测机制来保持有效的会话状态。通过这个连接，客户端可以发送请求并接收响应，同时也可以接收到 Watch 事件的通知。

关于会话中另外一个核心的概念是 sessionTimeOut(会话超时时间)，当由于网络故障或者客户端主动断开等原因，导致连接断开，此时只要在会话超时时间之内重新建立连接，则之前创建的会话依然有效。

#### 3.3 数据节点

Zookeeper 数据模型是由一系列基本数据单元 `Znode`(数据节点) 组成的节点树，其中根节点为 `/`。每个节点上都会保存自己的数据和节点信息。Zookeeper 中节点可以分为两大类：

+ **持久节点** ：节点一旦创建，除非被主动删除，否则一直存在；
+ **临时节点** ：一旦创建该节点的客户端会话失效，则所有该客户端创建的临时节点都会被删除。

临时节点和持久节点都可以添加一个特殊的属性：`SEQUENTIAL`，代表该节点是否具有递增属性。如果指定该属性，那么在这个节点创建时，Zookeeper 会自动在其节点名称后面追加一个由父节点维护的递增数字。

#### 3.4 节点信息

每个 ZNode 节点在存储数据的同时，都会维护一个叫做 `Stat` 的数据结构，里面存储了关于该节点的全部状态信息。如下：

| **状态属性**   | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| czxid          | 数据节点创建时的事务 ID                                      |
| ctime          | 数据节点创建时的时间                                         |
| mzxid          | 数据节点最后一次更新时的事务 ID                              |
| mtime          | 数据节点最后一次更新时的时间                                 |
| pzxid          | 数据节点的子节点最后一次被修改时的事务 ID                    |
| cversion       | 子节点的更改次数                                             |
| version        | 节点数据的更改次数                                           |
| aversion       | 节点的 ACL 的更改次数                                        |
| ephemeralOwner | 如果节点是临时节点，则表示创建该节点的会话的 SessionID；如果节点是持久节点，则该属性值为 0 |
| dataLength     | 数据内容的长度                                               |
| numChildren    | 数据节点当前的子节点个数                                     |

#### 3.5 Watcher

Zookeeper 中一个常用的功能是 Watcher(事件监听器)，它允许用户在指定节点上针对感兴趣的事件注册监听，当事件发生时，监听器会被触发，并将事件信息推送到客户端。该机制是 Zookeeper 实现分布式协调服务的重要特性。

#### 3.6 ACL

Zookeeper 采用 ACL(Access Control Lists) 策略来进行权限控制，类似于 UNIX 文件系统的权限控制。它定义了如下五种权限：

- **CREATE**：允许创建子节点；
- **READ**：允许从节点获取数据并列出其子节点；
- **WRITE**：允许为节点设置数据；
- **DELETE**：允许删除子节点；
- **ADMIN**：允许为节点设置权限。  



### ZAB协议

#### 4.1 ZAB协议与数据一致性

ZAB 协议是 Zookeeper 专门设计的一种支持崩溃恢复的原子广播协议。通过该协议，Zookeepe 基于主从模式的系统架构来保持集群中各个副本之间数据的一致性。具体如下：

Zookeeper 使用一个单一的主进程来接收并处理客户端的所有事务请求，并采用原子广播协议将数据状态的变更以事务 Proposal 的形式广播到所有的副本进程上去。如下图：

![zookeeper-zkcomponents](/picture/pictures/zookeeper-zkcomponents.jpg)

具体流程如下：

所有的事务请求必须由唯一的 Leader 服务来处理，Leader 服务将事务请求转换为事务 Proposal，并将该 Proposal 分发给集群中所有的 Follower 服务。如果有半数的 Follower 服务进行了正确的反馈，那么 Leader 就会再次向所有的 Follower 发出 Commit 消息，要求将前一个 Proposal 进行提交。

#### 4.2  ZAB协议的内容

ZAB 协议包括两种基本的模式，分别是崩溃恢复和消息广播：

##### 1. 崩溃恢复

当整个服务框架在启动过程中，或者当 Leader 服务器出现异常时，ZAB 协议就会进入恢复模式，通过过半选举机制产生新的 Leader，之后其他机器将从新的 Leader 上同步状态，当有过半机器完成状态同步后，就退出恢复模式，进入消息广播模式。

##### 2. 消息广播

ZAB 协议的消息广播过程使用的是原子广播协议。在整个消息的广播过程中，Leader 服务器会每个事物请求生成对应的 Proposal，并为其分配一个全局唯一的递增的事务 ID(ZXID)，之后再对其进行广播。具体过程如下：

Leader 服务会为每一个 Follower 服务器分配一个单独的队列，然后将事务 Proposal 依次放入队列中，并根据 FIFO(先进先出) 的策略进行消息发送。Follower 服务在接收到 Proposal 后，会将其以事务日志的形式写入本地磁盘中，并在写入成功后反馈给 Leader 一个 Ack 响应。当 Leader 接收到超过半数 Follower 的 Ack 响应后，就会广播一个 Commit 消息给所有的 Follower 以通知其进行事务提交，之后 Leader 自身也会完成对事务的提交。而每一个 Follower 则在接收到 Commit 消息后，完成事务的提交。

![zookeeper-brocast](/picture/pictures/zookeeper-brocast.jpg)



### 典型应用场景

#### 5.1数据的发布/订阅

数据的发布/订阅系统，通常也用作配置中心。在分布式系统中，你可能有成千上万个服务节点，如果想要对所有服务的某项配置进行更改，由于数据节点过多，你不可逐台进行修改，而应该在设计时采用统一的配置中心。之后发布者只需要将新的配置发送到配置中心，所有服务节点即可自动下载并进行更新，从而实现配置的集中管理和动态更新。

Zookeeper 通过 Watcher 机制可以实现数据的发布和订阅。分布式系统的所有的服务节点可以对某个 ZNode 注册监听，之后只需要将新的配置写入该 ZNode，所有服务节点都会收到该事件。

#### 5.2 命名服务

在分布式系统中，通常需要一个全局唯一的名字，如生成全局唯一的订单号等，Zookeeper 可以通过顺序节点的特性来生成全局唯一 ID，从而可以对分布式系统提供命名服务。

#### 5.3 Master选举

分布式系统一个重要的模式就是主从模式 (Master/Salves)，Zookeeper 可以用于该模式下的 Matser 选举。可以让所有服务节点去竞争性地创建同一个 ZNode，由于 Zookeeper 不能有路径相同的 ZNode，必然只有一个服务节点能够创建成功，这样该服务节点就可以成为 Master 节点。

#### 5.4 分布式锁

可以通过 Zookeeper 的临时节点和 Watcher 机制来实现分布式锁，这里以排它锁为例进行说明：

分布式系统的所有服务节点可以竞争性地去创建同一个临时 ZNode，由于 Zookeeper 不能有路径相同的 ZNode，必然只有一个服务节点能够创建成功，此时可以认为该节点获得了锁。其他没有获得锁的服务节点通过在该 ZNode 上注册监听，从而当锁释放时再去竞争获得锁。锁的释放情况有以下两种：

+ 当正常执行完业务逻辑后，客户端主动将临时 ZNode 删除，此时锁被释放；
+ 当获得锁的客户端发生宕机时，临时 ZNode 会被自动删除，此时认为锁已经释放。

当锁被释放后，其他服务节点则再次去竞争性地进行创建，但每次都只有一个服务节点能够获取到锁，这就是排他锁。

#### 5.5 集群管理

Zookeeper 还能解决大多数分布式系统中的问题：

+ 如可以通过创建临时节点来建立心跳检测机制。如果分布式系统的某个服务节点宕机了，则其持有的会话会超时，此时该临时节点会被删除，相应的监听事件就会被触发。
+ 分布式系统的每个服务节点还可以将自己的节点状态写入临时节点，从而完成状态报告或节点工作进度汇报。
+ 通过数据的订阅和发布功能，Zookeeper 还能对分布式系统进行模块的解耦和任务的调度。
+ 通过监听机制，还能对分布式系统的服务节点进行动态上下线，从而实现服务的动态扩容。

## ACL控制权限

### 前言

为了避免存储在 Zookeeper 上的数据被其他程序或者人为误修改，Zookeeper 提供了 ACL(Access Control Lists) 进行权限控制。只有拥有对应权限的用户才可以对节点进行增删改查等操作。下文分别介绍使用原生的 Shell 命令和 Apache Curator 客户端进行权限设置。

### 使用Shell进行权限管理

#### 2.1 设置与查看权限

想要给某个节点设置权限 (ACL)，有以下两个可选的命令：

```shell
 # 1.给已有节点赋予权限
 setAcl path acl
 
 # 2.在创建节点时候指定权限
 create [-s] [-e] path data acl
```

查看指定节点的权限命令如下：

```shell
getAcl path
```

#### 2.2 权限组成

Zookeeper 的权限由[scheme : id :permissions]三部分组成，其中 Schemes 和 Permissions 内置的可选项分别如下：

**Permissions 可选项**：

- **CREATE**：允许创建子节点；
- **READ**：允许从节点获取数据并列出其子节点；
- **WRITE**：允许为节点设置数据；
- **DELETE**：允许删除子节点；
- **ADMIN**：允许为节点设置权限。  

**Schemes 可选项**：

- **world**：默认模式，所有客户端都拥有指定的权限。world 下只有一个 id 选项，就是 anyone，通常组合写法为 `world:anyone:[permissons]`；
- **auth**：只有经过认证的用户才拥有指定的权限。通常组合写法为 `auth:user:password:[permissons]`，使用这种模式时，你需要先进行登录，之后采用 auth 模式设置权限时，`user` 和 `password` 都将使用登录的用户名和密码；
- **digest**：只有经过认证的用户才拥有指定的权限。通常组合写法为 `auth:user:BASE64(SHA1(password)):[permissons]`，这种形式下的密码必须通过 SHA1 和 BASE64 进行双重加密；
- **ip**：限制只有特定 IP 的客户端才拥有指定的权限。通常组成写法为 `ip:182.168.0.168:[permissions]`；
- **super**：代表超级管理员，拥有所有的权限，需要修改 Zookeeper 启动脚本进行配置。



#### 2.3 添加认证信息

可以使用如下所示的命令为当前 Session 添加用户认证信息，等价于登录操作。

```shell
# 格式
addauth scheme auth 

#示例：添加用户名为heibai,密码为root的用户认证信息
addauth digest heibai:root 
```



#### 2.4 权限设置示例

##### 1. world模式

world 是一种默认的模式，即创建时如果不指定权限，则默认的权限就是 world。

```shell
[zk: localhost:2181(CONNECTED) 32] create /hadoop 123
Created /hadoop
[zk: localhost:2181(CONNECTED) 33] getAcl /hadoop
'world,'anyone    #默认的权限
: cdrwa
[zk: localhost:2181(CONNECTED) 34] setAcl /hadoop world:anyone:cwda   # 修改节点，不允许所有客户端读
....
[zk: localhost:2181(CONNECTED) 35] get /hadoop
Authentication is not valid : /hadoop     # 权限不足

```

##### 2. auth模式

```shell
[zk: localhost:2181(CONNECTED) 36] addauth digest heibai:heibai  # 登录
[zk: localhost:2181(CONNECTED) 37] setAcl /hadoop auth::cdrwa    # 设置权限
[zk: localhost:2181(CONNECTED) 38] getAcl /hadoop                # 获取权限
'digest,'heibai:sCxtVJ1gPG8UW/jzFHR0A1ZKY5s=   #用户名和密码 (密码经过加密处理)，注意返回的权限类型是 digest
: cdrwa

#用户名和密码都是使用登录的用户名和密码，即使你在创建权限时候进行指定也是无效的
[zk: localhost:2181(CONNECTED) 39] setAcl /hadoop auth:root:root:cdrwa    #指定用户名和密码为 root
[zk: localhost:2181(CONNECTED) 40] getAcl /hadoop
'digest,'heibai:sCxtVJ1gPG8UW/jzFHR0A1ZKY5s=  #无效，使用的用户名和密码依然还是 heibai
: cdrwa

```

##### 3. digest模式

```shell
[zk:44] create /spark "spark" digest:heibai:sCxtVJ1gPG8UW/jzFHR0A1ZKY5s=:cdrwa  #指定用户名和加密后的密码
[zk:45] getAcl /spark  #获取权限
'digest,'heibai:sCxtVJ1gPG8UW/jzFHR0A1ZKY5s=   # 返回的权限类型是 digest
: cdrwa
```

到这里你可以发现使用 `auth` 模式设置的权限和使用 `digest` 模式设置的权限，在最终结果上，得到的权限模式都是 `digest`。某种程度上，你可以把 `auth` 模式理解成是 `digest` 模式的一种简便实现。因为在 `digest` 模式下，每次设置都需要书写用户名和加密后的密码，这是比较繁琐的，采用 `auth` 模式就可以避免这种麻烦。

##### 4. ip模式

限定只有特定的 ip 才能访问。

```shell
[zk: localhost:2181(CONNECTED) 46] create  /hive "hive" ip:192.168.0.108:cdrwa  
[zk: localhost:2181(CONNECTED) 47] get /hive
Authentication is not valid : /hive  # 当前主机已经不能访问
```

这里可以看到当前主机已经不能访问，想要能够再次访问，可以使用对应 IP 的客户端，或使用下面介绍的 `super` 模式。

##### 5. super模式

需要修改启动脚本 `zkServer.sh`，并在指定位置添加超级管理员账户和密码信息：

```shell
"-Dzookeeper.DigestAuthenticationProvider.superDigest=heibai:sCxtVJ1gPG8UW/jzFHR0A1ZKY5s=" 
```

![zookeeper-super](/picture/pictures/zookeeper-super.png)

修改完成后需要使用 `zkServer.sh restart` 重启服务，此时再次访问限制 IP 的节点：

```shell
[zk: localhost:2181(CONNECTED) 0] get /hive  #访问受限
Authentication is not valid : /hive
[zk: localhost:2181(CONNECTED) 1] addauth digest heibai:heibai  # 登录 (添加认证信息)
[zk: localhost:2181(CONNECTED) 2] get /hive  #成功访问
hive
cZxid = 0x158
ctime = Sat May 25 09:11:29 CST 2019
mZxid = 0x158
mtime = Sat May 25 09:11:29 CST 2019
pZxid = 0x158
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 0
```

### 使用Java客户端进行权限管理

#### 3.1 主要依赖

这里以 Apache Curator 为例，使用前需要导入相关依赖，完整依赖如下：

```xml
<dependencies>
    <!--Apache Curator 相关依赖-->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>4.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>4.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.13</version>
    </dependency>
    <!--单元测试相关依赖-->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>
```

#### 3.2 权限管理API

 Apache Curator 权限设置的示例如下：

```java
public class AclOperation {

    private CuratorFramework client = null;
    private static final String zkServerPath = "192.168.0.226:2181";
    private static final String nodePath = "/hadoop/hdfs";

    @Before
    public void prepare() {
        RetryPolicy retryPolicy = new RetryNTimes(3, 5000);
        client = CuratorFrameworkFactory.builder()
                .authorization("digest", "heibai:123456".getBytes()) //等价于 addauth 命令
                .connectString(zkServerPath)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
                .namespace("workspace").build();
        client.start();
    }

    /**
     * 新建节点并赋予权限
     */
    @Test
    public void createNodesWithAcl() throws Exception {
        List<ACL> aclList = new ArrayList<>();
        // 对密码进行加密
        String digest1 = DigestAuthenticationProvider.generateDigest("heibai:123456");
        String digest2 = DigestAuthenticationProvider.generateDigest("ying:123456");
        Id user01 = new Id("digest", digest1);
        Id user02 = new Id("digest", digest2);
        // 指定所有权限
        aclList.add(new ACL(Perms.ALL, user01));
        // 如果想要指定权限的组合，中间需要使用 | ,这里的|代表的是位运算中的 按位或
        aclList.add(new ACL(Perms.DELETE | Perms.CREATE, user02));

        // 创建节点
        byte[] data = "abc".getBytes();
        client.create().creatingParentsIfNeeded()
                .withMode(CreateMode.PERSISTENT)
                .withACL(aclList, true)
                .forPath(nodePath, data);
    }


    /**
     * 给已有节点设置权限,注意这会删除所有原来节点上已有的权限设置
     */
    @Test
    public void SetAcl() throws Exception {
        String digest = DigestAuthenticationProvider.generateDigest("admin:admin");
        Id user = new Id("digest", digest);
        client.setACL()
                .withACL(Collections.singletonList(new ACL(Perms.READ | Perms.DELETE, user)))
                .forPath(nodePath);
    }

    /**
     * 获取权限
     */
    @Test
    public void getAcl() throws Exception {
        List<ACL> aclList = client.getACL().forPath(nodePath);
        ACL acl = aclList.get(0);
        System.out.println(acl.getId().getId() 
                           + "是否有删读权限:" + (acl.getPerms() == (Perms.READ | Perms.DELETE)));
    }

    @After
    public void destroy() {
        if (client != null) {
            client.close();
        }
    }
}
```

## 常用Shell命令

### 节点增删改查

#### 1.1 启动服务和连接服务

```shell
# 启动服务
bin/zkServer.sh start

#连接服务 不指定服务地址则默认连接到localhost:2181
zkCli.sh -server hadoop001:2181
```

#### 1.2 help命令

使用 `help` 可以查看所有命令及格式。

```
ZooKeeper -server host:port -client-configuration properties-file cmd args
        addWatch [-m mode] path # optional mode is one of [PERSISTENT, PERSISTENT_RECURSIVE] - default is PERSISTENT_RECURSIVE
        addauth scheme auth
        close
        config [-c] [-w] [-s]
        connect host:port
        create [-s] [-e] [-c] [-t ttl] path [data] [acl]
        delete [-v version] path
        deleteall path [-b batch size]
        delquota [-n|-b] path
        get [-s] [-w] path
        getAcl [-s] path
        getAllChildrenNumber path
        getEphemerals path
        history
        listquota path
        ls [-s] [-w] [-R] path
        printwatches on|off
        quit
        reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
        redo cmdno
        removewatches path [-c|-d|-a] [-l]
        set [-s] [-v version] path data
        setAcl [-s] [-v version] [-R] path acl
        setquota -n|-b val path
        stat [-w] path
        sync path
        version
```


#### 1.3 查看节点列表

查看节点列表有 `ls path` 和 `ls -s path` 两个命令，后者是前者的增强，不仅可以查看指定路径下的所有节点，还可以查看当前节点的信息。

```shell
[zk: localhost:2181(CONNECTED) 0] ls /
[zk: localhost:2181(CONNECTED) 1] ls -s /
[a0000000001, b0000000002, c0000000003, hadoop, zookeeper]
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x300000011
cversion = 9
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 7
```

#### 1.4 新增节点

```shell
create [-s] [-e] [-c] [-t ttl] path [data] [acl]   #其中-s 为有序节点，-e 临时节点
```

创建节点并写入数据：

```shell
create /hadoop 123456
```

创建有序节点，此时创建的节点名为指定节点名 + 自增序号：

```shell
[zk: localhost:2181(CONNECTED) 23] create -s /a  "aaa"
Created /a0000000022
[zk: localhost:2181(CONNECTED) 24] create -s /b  "bbb"
Created /b0000000023
[zk: localhost:2181(CONNECTED) 25] create -s /c  "ccc"
Created /c0000000024
```

创建临时节点，临时节点会在会话过期后被删除：

```shell
[zk: localhost:2181(CONNECTED) 26] create -e /tmp  "tmp"
Created /tmp
```

#### 1.5 查看节点

##### 1. 获取节点数据

```shell
# 格式
# get [-s] [-w] path
[zk: localhost:2181(CONNECTED) 24] get -s -w /hadoop
123456
cZxid = 0x200000002
ctime = Sat Nov 12 17:08:09 CST 2020
mZxid = 0x200000002
mtime = Sat Nov 12 17:08:09 CST 2020
pZxid = 0x200000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 6
numChildren = 0

[zk: localhost:2181(CONNECTED) 25] get -s -w /tmp
tmp
cZxid = 0x200000006
ctime = Sat Nov 12 17:10:45 CST 2021
mZxid = 0x200000006
mtime = Sat Nov 12 17:10:45 CST 2021
pZxid = 0x200000006
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x1000251b9bb0000
dataLength = 3
numChildren = 0
```

节点各个属性如下表。其中一个重要的概念是 Zxid(ZooKeeper Transaction  Id)，ZooKeeper 节点的每一次更改都具有唯一的 Zxid，如果 Zxid1 小于 Zxid2，则 Zxid1 的更改发生在 Zxid2 更改之前。

| **状态属性**   | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| cZxid          | 数据节点创建时的事务 ID                                      |
| ctime          | 数据节点创建时的时间                                         |
| mZxid          | 数据节点最后一次更新时的事务 ID                              |
| mtime          | 数据节点最后一次更新时的时间                                 |
| pZxid          | 数据节点的子节点最后一次被修改时的事务 ID                    |
| cversion       | 子节点的更改次数                                             |
| dataVersion    | 节点数据的更改次数                                           |
| aclVersion     | 节点的 ACL 的更改次数                                        |
| ephemeralOwner | 如果节点是临时节点，则表示创建该节点的会话的 SessionID；如果节点是持久节点，则该属性值为 0 |
| dataLength     | 数据内容的长度                                               |
| numChildren    | 数据节点当前的子节点个数                                     |

##### 2. 查看节点状态

可以使用 `stat` 命令查看节点状态，它的返回值和 `get` 命令类似，但不会返回节点数据。

```shell
[zk: localhost:2181(CONNECTED) 32] stat /hadoop
cZxid = 0x14b
ctime = Fri May 24 17:03:06 CST 2019
mZxid = 0x14b
mtime = Fri May 24 17:03:06 CST 2019
pZxid = 0x14b
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 6
numChildren = 0
```

#### 1.6 更新节点

更新节点的命令是 `set`，可以直接进行修改，如下：

```shell
[zk: localhost:2181(CONNECTED) 33] set /hadoop 345
cZxid = 0x14b
ctime = Fri May 24 17:03:06 CST 2019
mZxid = 0x14c
mtime = Fri May 24 17:13:05 CST 2019
pZxid = 0x14b
cversion = 0
dataVersion = 1  # 注意更改后此时版本号为 1，默认创建时为 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
```

也可以基于版本号进行更改，此时类似于乐观锁机制，当你传入的数据版本号 (dataVersion) 和当前节点的数据版本号不符合时，zookeeper 会拒绝本次修改：

```shell
[zk: localhost:2181(CONNECTED) 34] set  -v 0 /hadoop 678
version No is not valid : /hadoop  #无效的版本号
```

#### 1.7 删除节点

删除节点的语法如下：

```shell
delete path -v [version]
```

和更新节点数据一样，也可以传入版本号，当你传入的数据版本号 (dataVersion) 和当前节点的数据版本号不符合时，zookeeper 不会执行删除操作。

```
[zk: localhost:2181(CONNECTED) 35] delete /hadoop -v 0
version No is not valid : /hadoop
[zk: localhost:2181(CONNECTED) 36] delete /hadoop
```

要想删除某个节点及其所有后代节点，可以使用递归删除，命令为 `rmr path`。

### 监听器

#### 2.1 get [-w] path

使用 `get [-w] path` 注册的监听器能够在节点内容发生改变的时候，向客户端发出通知。需要注意的是 zookeeper 的触发器是一次性的 (One-time trigger)，即触发一次后就会立即失效。

```shell
[zk: localhost:2181(CONNECTED) 4] get -w /hadoop
[zk: localhost:2181(CONNECTED) 5] set /hadoop 456

WATCHER::

WatchedEvent state:SyncConnected type:NodeDataChanged path:/hadoop
```

#### 2.2 stat [-w] path 

使用 `stat [-w] path` 注册的监听器能够在节点状态发生改变的时候，向客户端发出通知。

```shell
[zk: localhost:2181(CONNECTED) 7] stat -w /hadoop
[zk: localhost:2181(CONNECTED) 8] set /hadoop 112233
WATCHER::

WatchedEvent state:SyncConnected type:NodeDataChanged path:/hadoop
```

#### 2.3 ls [-s] [-w] [-R] path

使用 `ls [-s] [-w] -R path` 注册的监听器能够监听该节点下所有**子节点**的增加和删除操作。

```shell
[zk: localhost:2181(CONNECTED) 9] ls -w -R /hadoop
[]
[zk: localhost:2181(CONNECTED) 10] create  /hadoop/yarn "aaa"
WATCHER::

WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/hadoop
```



### zookeeper 四字命令

| 命令 | 功能描述                                                     |
| ---- | ------------------------------------------------------------ |
| conf | 打印有关服务配置的详细信息。                                 |
| cons | 列出连接到此服务器的所有客户端的完整连接/会话详细信息。包括有关接收/发送的数据包数量、会话 ID、操作延迟、上次执行的操作等信息... |
| crst | 重置所有连接的连接/会话统计信息。                            |
| dump | 列出未完成的会话和临时节点。                                 |
| envi | 打印有关服务环境的详细信息                                   |
| ruok | 测试服务器是否在非错误状态下运行。如果服务器正在运行，它将以 imok 响应。否则它根本不会响应。“imok”响应并不一定表示服务器已加入仲裁，只是服务器进程处于活动状态并绑定到指定的客户端端口。使用“stat”获取有关状态wrt quorum和客户端连接信息的详细信息。 |
| srst | 重置服务器统计信息。                                         |
| srvr | 列出服务器的完整详细信息。                                   |
| stat | 列出服务器和连接客户端的简要详细信息。                       |
| wchs | 列出有关服务器监视的简要信息。                               |
| wchc | 按会话列出有关服务器监视的详细信息。这将输出具有关联手表（路径）的会话（连接）列表。请注意，根据观察次数，此操作可能会很昂贵（即影响服务器性能），请谨慎使用。 |
| dirs | 以字节为单位显示快照和日志文件的总大小                       |
| wchp | 按路径列出有关服务器监视的详细信息。这将输出具有关联会话的路径（znode）列表。请注意，根据观察次数，此操作可能会很昂贵（即影响服务器性能），请谨慎使用。 |
| mntr | 输出可用于监控集群健康状况的变量列表。                       |
| isro | 测试服务器是否以只读模式运行。如果处于只读模式，服务器将响应“ro”，如果不是只读模式，则响应“rw”。 |
| hash | 返回与 zxid 关联的树摘要的最新历史记录。                     |
| gtmk | 以十进制格式的 64 位有符号长值形式获取当前跟踪掩码。有关stmk可能值的说明，请参见。 |
| stmk | 设置当前跟踪掩码。跟踪掩码是 64 位，其中每一位启用或禁用服务器上特定类别的跟踪日志记录。Log4J 必须首先配置为启用TRACE级别才能查看跟踪日志消息。跟踪掩码的位对应于以下跟踪记录类别。 |


> 更多四字命令可以参阅官方文档：https://zookeeper.apache.org/doc/current/zookeeperAdmin.html

使用前需要使用 `sudo yum install nc` 安装 nc 命令，使用示例如下：

```shell
[root@hadoop001 bin]# echo stat | nc localhost 2181
Zookeeper version: 3.4.13-2d71af4dbe22557fda74f9a9b4309b15a7487f03, 
built on 06/29/2018 04:05 GMT
Clients:
 /0:0:0:0:0:0:0:1:50584[1](queued=0,recved=371,sent=371)
 /0:0:0:0:0:0:0:1:50656[0](queued=0,recved=1,sent=0)
Latency min/avg/max: 0/0/19
Received: 372
Sent: 371
Connections: 2
Outstanding: 0
Zxid: 0x150
Mode: standalone
Node count: 167
```


```shell
[hadoop@node02 ~]$ echo stat | nc localhost 2181
stat is not executed because it is not in the whitelist.
```

- 若如上提示 指令不在白名单中，则需要修改`$ZK_HOME/conf/zoo.cfg`


```conf
vim $ZK_HOME/conf/zoo.cfg

# 将需要的命令添加到白名单中
4lw.commands.whitelist=stat, ruok, conf, isro

# 将所有命令添加到白名单中
4lw.commands.whitelist=*
```

- node01修改后将文件同步至其他节点

```shell
cd $ZK_HOME/conf
scp zoo.cfg node02:$PWD
scp zoo.cfg node03:$PWD
```

- 重启zookeeper



## Java客户端Curator

### 基本依赖

Curator 是 Netflix 公司开源的一个 Zookeeper 客户端，目前由 Apache 进行维护。与 Zookeeper 原生客户端相比，Curator 的抽象层次更高，功能也更加丰富，是目前 Zookeeper 使用范围最广的 Java 客户端。本篇文章主要讲解其基本使用，项目采用 Maven 构建，以单元测试的方法进行讲解，相关依赖如下：

```xml
<dependencies>
    <!--Curator 相关依赖-->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>4.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>4.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.13</version>
    </dependency>
    <!--单元测试相关依赖-->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>
```

### 客户端相关操作

#### 创建客户端实例

这里使用 `@Before` 在单元测试执行前创建客户端实例，并使用 `@After` 在单元测试后关闭客户端连接。

```java
public class BasicOperation {

    private CuratorFramework client = null;
    private static final String zkServerPath = "192.168.0.226:2181";
    private static final String nodePath = "/hadoop/yarn";

    @Before
    public void prepare() {
        // 重试策略
        RetryPolicy retryPolicy = new RetryNTimes(3, 5000);
        client = CuratorFrameworkFactory.builder()
        .connectString(zkServerPath)
        .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
        .namespace("workspace").build();  //指定命名空间后，client 的所有路径操作都会以/workspace 开头
        client.start();
    }

    @After
    public void destroy() {
        if (client != null) {
            client.close();
        }
    }
}
```

#### 重试策略

在连接 Zookeeper 时，Curator 提供了多种重试策略以满足各种需求，所有重试策略均继承自 `RetryPolicy` 接口，如下图：

![curator-retry-policy](/picture\pictures\curator-retry-policy.png)

这些重试策略类主要分为以下两类：

+ **RetryForever** ：代表一直重试，直到连接成功；
+ **SleepingRetry** ： 基于一定间隔时间的重试。这里以其子类 `ExponentialBackoffRetry` 为例说明，其构造器如下：

```java
/**
 * @param baseSleepTimeMs 重试之间等待的初始时间
 * @param maxRetries 最大重试次数
 * @param maxSleepMs 每次重试间隔的最长睡眠时间（毫秒）
 */
ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries, int maxSleepMs)    
```

#### 判断服务状态

```scala
@Test
public void getStatus() {
    CuratorFrameworkState state = client.getState();
    System.out.println("服务是否已经启动:" + (state == CuratorFrameworkState.STARTED));
}
```

### 节点增删改查

#### 创建节点

```java
@Test
public void createNodes() throws Exception {
    byte[] data = "abc".getBytes();
    client.create().creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT)      //节点类型
            .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
            .forPath(nodePath, data);
}
```

创建时可以指定节点类型，这里的节点类型和 Zookeeper 原生的一致，全部类型定义在枚举类 `CreateMode` 中：

```java
public enum CreateMode {
    // 永久节点
    PERSISTENT (0, false, false),
    //永久有序节点
    PERSISTENT_SEQUENTIAL (2, false, true),
    // 临时节点
    EPHEMERAL (1, true, false),
    // 临时有序节点
    EPHEMERAL_SEQUENTIAL (3, true, true);
    ....
}
```

#### 获取节点信息

```scala
@Test
public void getNode() throws Exception {
    Stat stat = new Stat();
    byte[] data = client.getData().storingStatIn(stat).forPath(nodePath);
    System.out.println("节点数据:" + new String(data));
    System.out.println("节点信息:" + stat.toString());
}
```

如上所示，节点信息被封装在 `Stat` 类中，其主要属性如下：

```java
public class Stat implements Record {
    private long czxid;
    private long mzxid;
    private long ctime;
    private long mtime;
    private int version;
    private int cversion;
    private int aversion;
    private long ephemeralOwner;
    private int dataLength;
    private int numChildren;
    private long pzxid;
    ...
}
```

每个属性的含义如下：

| **状态属性**   | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| czxid          | 数据节点创建时的事务 ID                                      |
| ctime          | 数据节点创建时的时间                                         |
| mzxid          | 数据节点最后一次更新时的事务 ID                              |
| mtime          | 数据节点最后一次更新时的时间                                 |
| pzxid          | 数据节点的子节点最后一次被修改时的事务 ID                    |
| cversion       | 子节点的更改次数                                             |
| version        | 节点数据的更改次数                                           |
| aversion       | 节点的 ACL 的更改次数                                        |
| ephemeralOwner | 如果节点是临时节点，则表示创建该节点的会话的 SessionID；如果节点是持久节点，则该属性值为 0 |
| dataLength     | 数据内容的长度                                               |
| numChildren    | 数据节点当前的子节点个数                                     |

#### 获取子节点列表

```java
@Test
public void getChildrenNodes() throws Exception {
    List<String> childNodes = client.getChildren().forPath("/hadoop");
    for (String s : childNodes) {
        System.out.println(s);
    }
}
```

#### 更新节点

更新时可以传入版本号也可以不传入，如果传入则类似于乐观锁机制，只有在版本号正确的时候才会被更新。

```scala
@Test
public void updateNode() throws Exception {
    byte[] newData = "defg".getBytes();
    client.setData().withVersion(0)     // 传入版本号，如果版本号错误则拒绝更新操作,并抛出 BadVersion 异常
            .forPath(nodePath, newData);
}
```

#### 删除节点

```java
@Test
public void deleteNodes() throws Exception {
    client.delete()
            .guaranteed()                // 如果删除失败，那么在会继续执行，直到成功
            .deletingChildrenIfNeeded()  // 如果有子节点，则递归删除
            .withVersion(0)              // 传入版本号，如果版本号错误则拒绝删除操作,并抛出 BadVersion 异常
            .forPath(nodePath);
}
```

#### 判断节点是否存在

```java
@Test
public void existNode() throws Exception {
    // 如果节点存在则返回其状态信息如果不存在则为 null
    Stat stat = client.checkExists().forPath(nodePath + "aa/bb/cc");
    System.out.println("节点是否存在:" + !(stat == null));
}
```



### 监听事件

#### 创建一次性监听

和 Zookeeper 原生监听一样，使用 `usingWatcher` 注册的监听是一次性的，即监听只会触发一次，触发后就销毁。示例如下：

```java
@Test
public void DisposableWatch() throws Exception {
    client.getData().usingWatcher(new CuratorWatcher() {
        public void process(WatchedEvent event) {
            System.out.println("节点" + event.getPath() + "发生了事件:" + event.getType());
        }
    }).forPath(nodePath);
    Thread.sleep(1000 * 1000);  //休眠以观察测试效果
}
```

#### 创建永久监听

Curator 还提供了创建永久监听的 API，其使用方式如下：

```java
@Test
public void permanentWatch() throws Exception {
    // 使用 NodeCache 包装节点，对其注册的监听作用于节点，且是永久性的
    NodeCache nodeCache = new NodeCache(client, nodePath);
    // 通常设置为 true, 代表创建 nodeCache 时,就去获取对应节点的值并缓存
    nodeCache.start(true);
    nodeCache.getListenable().addListener(new NodeCacheListener() {
        public void nodeChanged() {
            ChildData currentData = nodeCache.getCurrentData();
            if (currentData != null) {
                System.out.println("节点路径：" + currentData.getPath() +
                        "数据：" + new String(currentData.getData()));
            }
        }
    });
    Thread.sleep(1000 * 1000);  //休眠以观察测试效果
}
```

#### 监听子节点

这里以监听 `/hadoop` 下所有子节点为例，实现方式如下：

@Test
public void permanentChildrenNodesWatch() throws Exception {

    // 第三个参数代表除了节点状态外，是否还缓存节点内容
    PathChildrenCache childrenCache = new PathChildrenCache(client, "/hadoop", true);
    /*
         * StartMode 代表初始化方式:
         *    NORMAL: 异步初始化
         *    BUILD_INITIAL_CACHE: 同步初始化
         *    POST_INITIALIZED_EVENT: 异步并通知,初始化之后会触发 INITIALIZED 事件
         */
    childrenCache.start(StartMode.POST_INITIALIZED_EVENT);
    
    List<ChildData> childDataList = childrenCache.getCurrentData();
    System.out.println("当前数据节点的子节点列表：");
    childDataList.forEach(x -> System.out.println(x.getPath()));
    
    childrenCache.getListenable().addListener(new PathChildrenCacheListener() {
        public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) {
            switch (event.getType()) {
                case INITIALIZED:
                System.out.println("childrenCache 初始化完成");
                break;
                case CHILD_ADDED:
                // 需要注意的是: 即使是之前已经存在的子节点，也会触发该监听，因为会把该子节点加入 childrenCache 缓存中
                System.out.println("增加子节点:" + event.getData().getPath());
                break;
                case CHILD_REMOVED:
                System.out.println("删除子节点:" + event.getData().getPath());
                break;
                case CHILD_UPDATED:
                System.out.println("被修改的子节点的路径:" + event.getData().getPath());
                System.out.println("修改后的数据:" + new String(event.getData().getData()));
                break;
            }
        }
    });
    Thread.sleep(1000 * 1000); //休眠以观察测试效果
    }
