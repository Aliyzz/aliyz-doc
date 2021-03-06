# Redis 从认识开始

-----

>***"Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker."*** —— Redis是一个开放源代码（BSD许可）的内存中数据结构存储，用作数据库，缓存和消息代理。(摘自官网)<br>
>
>它是一个开源，高级的键值存储和一个适用的解决方案，用于构建高性能，可扩展的 Web 应用程序。Redis 也被作者戏称为 **数据结构服务器** ，这意味着使用者可以通过一些命令，基于带有 TCP 套接字的简单 服务器-客户端 协议来访问一组 可变数据结构 (*在 Redis 中都采用键值对的方式，只不过对应的数据结构不一样罢了*)。

## *What is Redis?*

- Redis是一个开源的内存中的数据结构存储系统，它可以用作：数据库、缓存和消息中间件；采用的是基于内存的采用的是单进程单线程模型的 KV 数据库，由C语言编写。

- 它支持多种类型的数据结构，如字符串（Strings），散列（Hash），列表（List），集合（Set），有序集合（Sorted Set或者是ZSet）与范围查询，Bitmaps，Hyperloglogs 和地理空间（Geospatial）索引半径查询。其中常见的数据结构类型有：String、List、Set、Hash、ZSet这5种。

- Redis 内置了复制（Replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（Transactions） 和不同级别的磁盘持久化（Persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（High Availability）。

- Redis也提供了持久化的选项，这些选项可以让用户将自己的数据保存到磁盘上面进行存储。根据实际情况，可以每隔一定时间将数据集导出到磁盘（快照），或者追加到命令日志中（AOF只追加文件），他会在执行写命令时，将被执行的写命令复制到硬盘里面。您也可以关闭持久化功能，将Redis作为一个高效的网络的缓存数据功能使用。

- Redis不使用表，他的数据库不会预定义或者强制去要求用户对Redis存储的不同数据进行关联。

- 数据库的工作模式按存储方式可分为：硬盘数据库和内存数据库。Redis 将数据储存在内存里面，读写数据的时候都不会受到硬盘 I/O 速度的限制，所以速度极快。

## *Why is Redis?*

- ***异常快*** - Redis 非常快，每秒可执行大约 110000 次的设置(SET)操作，每秒大约可执行 81000 次的读取/获取(GET)操作。
- ***支持丰富的数据类型*** - Redis 支持开发人员常用的大多数数据类型，例如列表，集合，排序集和散列等等。这使得 Redis 很容易被用来解决各种问题，因为我们知道哪些问题可以更好使用地哪些数据类型来处理解决。
- ***操作具有原子性*** - 所有 Redis 操作都是原子操作，这确保如果两个客户端并发访问，Redis 服务器能接收更新的值。
- ***多实用工具*** - Redis 是一个多实用工具，可用于多种用例，如：缓存，消息队列(Redis 本地支持发布/订阅)，应用程序中的任何短期数据，例如，web应用程序中的会话，网页命中计数等。

## Why is Redis so fast

1. 完全基于内存，绝大部分请求是纯粹的内存操作，非常快速。数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；

2. 数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的；

3. 采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；

4. 使用多路I/O复用模型，非阻塞IO；

5. 使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；
