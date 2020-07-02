# 干货 ZooKeeper

-----

#### 感觉一波

> 基于 ***CP*** 模型的服务发现中间件，服务于分布式系统，可以做 ***统一配置管理***、***统一命名服务***、***分布式锁***、***集群管理*** 。
> 
> 一句话描述就是：基于 ***`监听 + Znode节点(持久/临时 * 普通/有序)`*** 变换玩转。

#### 数据结构

> 跟Unix文件系统非常类似，可以看做是一颗树，每个节点叫做 `ZNode`，每一个节点可以通过路径来标识，如：“/”、“/java/zk”。
> 
> 节点类型：
> 
>> ***临时(Ephemeral)***：当客户端和服务端断开连接后，所创建的Znode(节点)会自动删除 <br>
>> ***持久(Persistent)***：当客户端和服务端断开连接后，所创建的Znode(节点)不会删除
>>> 每个节点也分 `普通节点` 和 `有序节点`

#### 监听器

> 监听Znode节点的 **数据变化** <br>
> 监听子节点的 **增减变化**

#### 使用场景

> 统一配置管理
>> 多个系统将公共的配置文件维护到 ZK 的指定 ZNode，系统监听这个节点来同步配置的变更
>
> 统一命名服务
>> 一个域名对用多个IP，将域名和IP的关系维护到 ZK 的指定 ZNode，用户就可以通过域名访问资源了
>
> 分布式锁
>> 多个应用系统都访问 ZK 的 `/locks`（可以是其他） 节点，并创建 ***带顺序号的临时（EPHEMERAL_SEQUENTIAL）*** 节点，每个系统获得 /locks 下所有子节点，判断自己创建的是不是 ***最小*** 的那个节点，如果是，获得锁；如果不是，则监听比自己要小 `1` 的节点变化；
>>> 释放锁：执行完操作后，把创建的节点给删掉
>
> 集群管理
>> 集群中的成员都访问 ZK 的 `/groupMember`（可以是其他） 节点，并创建 ***临时节点***，通过监听 /groupMember 节点下的子节点，感知成员上下线。
>>> 动态选举 master，将 Znode 节点类型改成 ***带顺序号的临时节点***，ZK 会每次选举最小编号的作为 master。

#### ZAB协议

> Zab（Zookeeper Atomic Broadcast）是为ZooKeeper协设计的崩溃恢复原子广播协议，它保证zookeeper集群数据的一致性和命令的全局有序性。

- 集群角色

> ***Leader***：同一时间集群总只允许有一个 Leader，提供对客户端的读写功能，负责将数据同步至各个节点；<br>
> ***Follower***：提供对客户端读功能，写请求则转发给 Leader 处理，当 Leader 崩溃失联之后参与 Leader 选举；<br>
> ***Observer***：与 Follower 不同的是但不参与 Leader 选举。

- 服务状态

> Zookeeper是通过自身的状态来区分自己所属的角色，来执行自己应该的任务：
> 
> ***LOOKING***：当节点认为群集中没有 Leader，服务器会进入 LOOKING 状态，目的是为了查找或者选举 Leader；<br>
> ***FOLLOWING***：follower角色；<br>
> ***LEADING***：leader角色；<br>
> ***OBSERVING***：observer角色；

- ZAB 状态

> Zookeeper还给ZAB定义的4中状态，反应Zookeeper从选举到对外提供服务的过程中的四个步骤：
> 
> ***ELECTION***: 集群进入选举状态，此过程会选出一个节点作为 leader 角色；<br>
> ***DISCOVERY***：连接上 leader，响应 leader 心跳，并且检测 leader 的角色是否更改，通过此步骤之后选举出的 leader 才能执行真正职务；<br>
> ***SYNCHRONIZATION***：整个集群都确认 leader 之后，将会把 leader 的数据同步到各个节点，保证整个集群的数据一致性；<br>
> ***BROADCAST***：过渡到广播状态，集群开始对外提供服务。

- ZXID

> 也就是事务 id， 为了保证事务的顺序一致性，采用了递增的事务 id 号（zxid）来标识事务，所有的提议（proposal）都在被提出的时候加上了 zxid。
> 
> `zxid = epoch（32位） + counter（32位）`
> 
>> ZAB协议通过 纪元（epoch） 编号来区分 Leader 周期变化的策略，用来标识 leader 关系是否改变，每次一个 leader 被选出来，它都会有一个新的 epoch =（原来的epoch+1），标识当前属于那个leader的 统治时期；<br>
>> 计数器（counter）是 ***递增的全局有序的*** 数字，当有事物提交时 +1，如：新增或删除一个ZNode。

- 选举

> ***选举发生的时机***
>> 1、集群启动，没有 leader；2、当前 leader 不可达（下线、断网）
>
> ***选举规则*** 

```java
/*
 * We return true if one of the following three cases hold:
 * 1- New epoch is higher
 * 2- New epoch is the same as current epoch, but new zxid is higher
 * 3- New epoch is the same as current epoch, new zxid is the same
 *  as current zxid, but server id is higher.
 */

return ((newEpoch > curEpoch)
        || ((newEpoch == curEpoch)
            && ((newZxid > curZxid)
                || ((newZxid == curZxid)
                    && (newId > curId)))));
```

> ***选举流程*** 
> 
> 1. 所有节点第一票先选举自己当leader，将投票信息广播出去；<br>
> 2. 从队列中接受投票信息；<br>
> 3. 按照规则判断是否需要更改投票信息，将更改后的投票信息再次广播出去；<br>
> 4. 判断是否有 **超过一半** 的投票选举同一个节点，如果是选举结束根据投票结果设置自己的服务状态，选举结束，否则继续进入投票流程。

- 广播

> zab在广播状态中保证以下特征：
> 
> 1. 可靠传递:  如果消息m由一台服务器传递，那么它最终将由所有服务器传递。
> 2. 全局有序: 如果一个消息a在消息b之前被一台服务器交付，那么所有服务器都交付了a和b，并且a先于b。
> 3. 因果有序: 如果消息a在因果上先于消息b并且二者都被交付，那么a必须排在b之前。
> 
> 当收到客户端的写请求的时候会经历以下几个步骤：
> 
> 1. Leader收到客户端的写请求，生成一个事务（Proposal），其中包含了zxid；
> 2. Leader开始广播该事务，需要注意的是所有节点的通讯都是由一个FIFO的队列维护的；
> 3. Follower接受到事务之后，将事务写入本地磁盘，写入成功之后返回Leader一个ACK；
> 4. Leader收到 **过半** 的ACK之后，开始提交本事务，并广播事务提交信息；
> 5. 从节点开始提交本事务。
> 
>> 第一，服务之前用 TCP 协议进行通讯，保证在网络传输中的有序性；<br>
>> 第二，节点之前都维护了一个 FIFO 的队列，保证全局有序性；<br>
>> 第三，通过全局递增的zxid保证因果有序性。

- ZAB 状态流转

> 1. 服务在启动或者和 leader 失联之后服务状态转为 LOOKING；<br>
> 2. 如果 leader 不存在选举 leader，如果存在直接连接 leader，此时 zab 协议状态为**`ELECTION`**；<br>
> 3. 如果有超过半数的投票选择同一台 server，则 leader 选举结束，被选举为 leader 的server 服务状态为 LEADING，其他 server 服务状态为 FOLLOWING/OBSERVING；<br>
> 4. 所有 server 连接上 leader，此时 zab 协议状态为 **`DISCOVERY`**；<br>
> 5. leader 同步数据给其他从节点，使各个从节点数据和 leader 保持一致，此时 zab 协议状态为 **`SYNCHRONIZATION`**；<br>
> 6. 同步超过一半的 server 之后，集群对外提供服务，此时 zab 状态为 **`BROADCAST`**。