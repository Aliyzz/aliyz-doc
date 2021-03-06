# Redis 集群&哨兵&分片

-----

> 详细参考：`https://www.cnblogs.com/huangfuyuan/p/9880379.html`

## Redis 分区（分片）

- 概念

> 分区是分割数据到多个Redis实例的处理过程，因此每个实例只保存key的一个子集。

- 优势

> 扩展存储空间 ***`内存`***、扩展计算能力 ***`CPU`***、扩展网络带宽 ***`IO`***。

- 不足

> 多key的操作通常不被支持，如：交集操作； <br>
> 多key的Redis事务不能使用； <br>
> 数据处理较为复杂，如：RDB/AOF； <br>
> 客户端、代理等实现分区架构，增加或删除容量也比较复杂。

- 类型

> ***范围分区***：映射一定范围的对象到特定的Redis实例；但需要维护要一个区间范围到实例的映射表。
> 
> ***哈希分区***：将 key 进行HASH运算（取模等），确定对象到特定的Redis实例。

## Redis cluster

- 概念

> 是Redis的分布式解决方案，能`自动分片`、`内置的高可用支持`、`支撑多 master 多 slave组合`、`自动选主`等。
> 
> 适合 `海量数据+高并发+高可用` 的场景
> 
> 默认情况下，redis cluster 的核心的理念，主要是用 slave 做高可用的，每个 master 挂一两个 slave，主要是做数据的热备，还有 master 故障时的主备切换，实现高可用的。

- Hash slot算法

```python
hash slot = CRC16(key) % 16384
```

> 集群中，每个 master 都会持有部分 slot，增加一个 master，就将其他 master 的 hash slot 移动部分过去，减少一个 master，就将它的 hash slot 移动到其他 master 上去；并且移动 hash slot 的成本是非常低的。
> 
> 对指定的数据，可以让他们走同一个 hash slot ，通过 hash tag 来实现。
> 
> 客户端向节点发送键命令，节点要计算这个键属于哪个槽。如果是自己负责这个槽，那么直接执行命令，如果不是，向客户端返回一个 ***MOVED*** 错误，指引客户端转向正确的节点。

- gossip通信协议

> **内容**：故障信息、节点的增加和移除、hash slot信息等
>
> 特点：集群元数据的更新比较分散，不集中在一个节点，更新请求陆陆续续，打到所有节点上去更新，有一定的延时，降低了压力; 这可能导致集群的一些操作会有一些滞后。
> 
> **原理**：<br>
>> meet：某个节点发送 meet 给新加入的节点，让其加入集群，然后新节点就会开始与其他节点进行通信 
>> 
>> ping：每个节点都会频繁给其他 **节点、集群** 发送ping，告知**自己状态**、**维护的集群元数据**，并 **更新、交换** 集群元数据等 <br>
>> 
>>> 频率：10次/s，每次会选择5个最久没有通信的其他节点；<br>
>>> 如果发现与某个节点通信延时达到了 ***`cluster_node_timeout / 2`***，那么立即发送 ping，避免出现严重的元数据不一致；<br>
>>> 每次ping，一个是带上自己节点的信息，还有就是带上 1/10 （即 [3, N-2]）其他节点的信息。
>>
>> pong：返回 ping 和 meet，包含自己的状态和其他信息，也可以用于信息广播和更新
>> 
>> fail：某个节点判断另一个节点fail之后，就发送fail给其他节点，通知其他节点，指定的节点宕机了

- 请求重定向

> 客户端可能会挑选 **任意** 一个 Redis 实例去发送命令，接收到命令后会计算 key 对应的 hash slot，如果在本地就在本地处理，否则返回 moved 给客户端，让客户端进行重定向；
> 
> 节点间通过gossip协议进行数据交换，就知道每个 hash slot 在哪个节点上。

- 高可用性与主备切换

> ***判断节点宕机***
> 
>> 主观宕机：一个节点认为另外一个节点宕机，***`cluster_state: pfail`***；<br>
>> 客观宕机：多个节点都认为另外一个节点宕机，***`cluster_state: fail`***；<br>
>> 在 cluster-node-timeout 内，某个节点一直没有返回 pong，***`cluster_state: pfail`***；<br>
>> 如果一个节点认为某个节点 pfail 了，那么会在 gossip ping消息给其他节点，如果超过半数 `(N/2 + 1)` 的节点都认为 pfail 了，那么就会变成 fail。
>
> ***从节点过滤***
> 
>> 对宕机的 master node，从其所有的 slave node 中，选择一个切换成 master node；<br>
>> 检查每个 slave node 与 master node 断开连接的时间，如果超过了 ***`cluster-node-timeout * cluster-slave-validity-factor`***，那么就没有资格切换成 master。
>
> ***从节点选举***
> 
>> **排序**：每个从节点，都根据自己对 master 复制数据的 offset，来设置一个选举时间，offset 越大 ***（复制数据越多）*** 的从节点，选举时间越靠前，优先进行选举（slave priority）；
>> 
>> **选举**：所有 ***master node*** 开始 slave 选举投票，如果大部分 master node `(N/2 + 1)` 都投票给了某个从节点，那么选举通过；
>> 
>> **换主**：从节点执行主备切换。

## Redis sentinal

- 概念

> 哨兵是一个独立的进程，其原理是哨兵通过发送命令，等待Redis服务器响应，**包括主服务器和从服务器**，从而监控运行的多个Redis实例的运行状态（哨兵之间也会相互监控）；

- 功能

> a. 监控(Monitoring)
> 
>> 哨兵(sentinel) 会不断地检查你的Master和Slave是否运作正常
>
> b. 提醒(Notification)
> 
>> 当被监控的某个 Redis出现问题时, 哨兵(sentinel) 可以通过 API 向管理员或者其他应用程序发送通知
>
> c. 自动故障切换（Automatic failover）
> 
>> **主观下线（S_DOWN）**：假设主服务器宕机，哨兵1先检测到这个结果，系统并不会马上进行failover过程，仅仅是哨兵1主观的认为主服务器不可用；
>>
>> **客观下线（O_DOWN）**：当后面的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次 *投票*，投票的结果由一个哨兵发起，进行failover操作。切换成功后，就会通过 *发布订阅模式*，让各个哨兵把自己监控的从服务器实现切换主机。
>> 
>>> ***选举（vote）*** 选点的依据依次是（序号越大，被选的可能性越大）：
>>> 
>>> 1. 网络连接正常
>>> 2. 5秒内回复过INFO命令：最近一次应答ping 时间不超过5倍ping间隔（默认1秒）
>>> 3. 10*down-after-milliseconds 内与主连接过的
>>> 4. 从服务器优先级 slave_priority `较低的`
>>> 5. 复制偏移量 offset `较大的`
>>> 6. runid `较小的`
>>> 
>>>> slave_priority：这个是在配置文件中指定，默认配置为100；<br>
>>>> replication offset：每个slave在与master同步后offset自动增；<br>
>>>> runid：每个redis实例，都会有一个runid,通常是一个40位的随机字符串,在redis启动时设置，重复概率非常小
>> 
>> 当客户端试图连接失效的 Master 时,集群也会向客户端返回新 Master 的地址，使得集群可以使用Master代替失效Master。