# Redis 基本数据类型

------

在redis中，所有数据类型都被封装在一个redisObject结构中，用于提供统一的接口。


## String

>最基本的数据类型，也是一种可修改的动态字符串，一个key对应一个value；可以包含任何数据，比如jpg图片或者序列化的对象；一个键最大能存储512MB

- 常用命令

	```python
	1.查询：GET key
	2.新增：SET key value
	3.设置过期：EXPIRE key seconds
	4.删除：DEL key
	```

- 使用场景
	- **基于token的单点登录**
	- **使用 `INCR` 和 `DECR` 实现分布式计数器**
	- **string + json 的对象存储**
	- **分布式锁**：基于`setnx`、`expire`、`del`组合实现
	

## Hash

>Hash是一个键值对集合，也叫字典或映射表，底层会有2种不同实现: 

> - Hash的成员比较少时Redis为了节省内存会采用*`类似一维数组`*的方式来紧凑存储，对应的value redisObject的 encoding=zipmap；
> - 当成员数量增大时会自动转成真正的HashMap，使用链地址法来解决部分**哈希冲突**；此时 encoding=ht

- 常用命令

	```python
	1.查询属性：HGET key field
	2.查询全部属性：HGETALL key
	3.新增单个属性：HSET key field value
	4.新增多个属性：HMSET key field1 value1 [field2 value2]
	5.删除属性：HDEL key field1 [field2]
	```

- 使用场景
	- **购物车**：以用户id为key，商品id为field，商品数量为value
	- **存储对象**(当对象的某个属性需要频繁修改时)：与string类型对象存储的区别
	
		|         | STRING + JSON      | HASH  
		| ------- | :----------------: |  :----:
		| 效率     | 很高               |  高  
		| 容量     | 低                 |  低  
		| 灵活性   | 低                 |  高  
		| 序列化   | 简单                |  复杂  

## List

>Redis最重要的数据结构之一，双向链表结构，这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)；可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

- 常用命令

	```python
	1.左阻塞取值（如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止，下同）：
	BLPOP key1 [key2] timeout
	2.右阻塞取值：BRPOP key1 [key2] timeout
	3.左/右非阻塞取值（最后一个元素）：LPOP/RPOP key
	4.左/右新增值：LPUSH/RPUSH key value1 [value2]
	5.获取指定范围内的元素：LRANGE key start stop
	```

- 使用场景
	- **消息栈**：基于`lpop`和`lpush`（或者 `rpush`和`rpop`）实现。
	- **消息队列**：基于`lpop`和`rpush`（或者 `lpush`和`rpop`）实现。
	- **排行榜**（只适合定时计算排行榜）：`lrange`命令可以分页查看队列中的数据，可将每隔一段时间计算一次的排行榜存储在list类型中，如每日的手机销量排行、学校月考学生的成绩排名等。
	- **最新列表**：`lpush`和`lrange`能实现最新列表的功能，每次通过lpush命令往列表里插入新的元素，然后通过lrange命令读取最新的元素列表，如朋友圈的点赞列表、评论列表。**对于频繁更新的列表不适合**。

## Set

>与List类似，但他支持自动排重，同时也提供了判断某个成员是否在一个set集合内的重要接口。

- 常用命令

	```python
	sadd,spop,smembers,sunion
	1.向key中放入元素：SADD key member1 [member2]
	2.移除并返回集合中的一个随机元素：SPOP key
	3.返回集合中的所有成员：SMEMBERS key
	4.返回所有给定集合的并集：SUNION key1 [key2]
	5.返回给定所有集合的差集：SDIFF key1 [key2]
	```

- 使用场景
	- **好友/关注/粉丝/感兴趣的人集合**
	
	```
	1.sinter命令可以获得A和B两个用户的共同好友
	2.sismember命令可以判断A是否是B的好友
	3.scard命令可以获取好友数量
	4.关注时，smove命令可以将B从A的粉丝集合转移到A的好友集合「」
	
	```

	- **随机展示**：如首页随机展示等，基于`srandmember`命令则可以从中随机获取几个。
	- **黑名单/白名单**：如用户黑名单、ip黑名单、设备黑名单等，`sismember`命令可用于判断用户、ip、设备是否处于黑名单之中。

## SortedSet

>Set不是自动有序的，而sorted set可以通过用户额外提供一个优先级(score)的参数来为成员实现自动排序。

>内部使用 `dict字典` 和 `跳跃表(SkipList)` 来保证数据的存储和有序，dict里放的是成员到score的映射，而跳跃表里存放的是所有的成员，排序依据是HashMap里存的score，使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单。

>***其实Redis有序列表有压缩列表ziplist（内存连续的特殊双向链表）和跳表skiplist两种实现方式，通过encoding识别，当数据项数目小于zset\_max\_ziplist\_entries(默认为128)，且保存的所有元素长度不超过zset\_max\_ziplist\_value(默认为64)时，则用ziplist实现有序集合，否则使用zset结构，zset底层使用skiplist跳表和dict字典***

- 常用命令

	```python
	1.添加（或更新分数）一个或多个成员：ZADD key score1 member1 [score2 member2]
	2.获取有序集合的成员数：ZCARD key
	3.返回指定索引区间内的成员：ZRANGE key start stop [WITHSCORES]
	4.移除一个或多个成员：ZREM key member [member ...]
	5.返回指定成员的索引：ZRANK key member
	```

- 使用场景

	- **构造延迟队列**
	- **排行榜**：如游戏排名、微博热点话题等
	- **最新列表**
	```
	根据数据量的多少
	```
	

## 说明

>对于`排行榜`和`最新列表`两种应用场景，list类型能做到的sorted set类型都能做到，因为SortedSet类型占用的内存容量是List类型的***`数倍之多`***，取舍还得根据具体情况。

>如果你用的是Redis Cluster集群，对于sinter、smove等这种操作多个key的命令，要求这两个key必须存储在同一个slot（槽位）中，否则会报出:
``` (error) CROSSSLOT Keys in request don't hash to the same slot ```

>Redis Cluster一共有 ***16384*** 个slot，每个key都是通过哈希算法CRC16(key)获取数值哈希，再模16384来定位slot的。

>要使得两个key处于同一slot，除了两个key一模一样，再有就是Redis提供了一种 **`Hash Tag`** 的功能，在key中使用 {} 括起key中的一部分，在进行 `CRC16(key) mod 16384` 的过程中，只会对{}内的字符串计算，这样两个 key 就会放到同一个槽中。
