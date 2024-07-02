
key-value的基于**内存**的NoSQL数据库
- **单线程**，每个命令具备***原子性***，支持事务
- 速度快。
- 支持数据持久化
- 支持主从集群/分片集群
- 支持多编程语言

#### 使用场景
1. 热点数据的缓存
2. 事件发布订阅
3. 分布式锁
4. 限时业务的运用
	- redis中可以使用`expire`命令设置一个键的*过期时间*，到期后redis会删除它
	- 利用这一特性可以运用在限时的优惠活动信息、手机验证码等业务场景。
5. 计数器
	- redis由于`incrby`命令可以实现**原子性的递增**
	- 高并发的秒杀活动、分布式序列号的生成
	- 限制批量发短信、接口请求/调用限制等等。

# 底层实现
redis完全基于内存，单线程，非阻塞多路I/O复用

## 单线程模型
避免了上下文切换和竞争条件，不存在多线程消耗CPU
不考虑锁 (加锁, 释放锁, 死锁)

缺点：
1. 大量写
2. [[#大Key问题]]
3. 高并发
4. 并发处理可能阻塞（持久化操作[[#RDB]]，但bgsave可以异步持久化

### 不是完全单线程
https://cloud.tencent.com/developer/article/2390685
##### 异步多线程
有一些多线程用于异步操作
![](https://developer.qcloudimg.com/http-save/yehe-10782502/bdc28073668317610963bf5f51384db8.png)
bio_close_file 异步删除文件
bio_aof 异步刷[[#AOF]]到磁盘
bio_lazy_free 异步删除数据（懒删除

##### I/O多线程
Redis 6.0加入了多线程I/O，并行读取客户端命令请求，并行向客户端返回结果。
![redisIO多线程](https://developer.qcloudimg.com/http-save/yehe-10782502/977d48108cc4282d6375548d3776ce00.png)

#### 异步多进程
持久化[[#RDB]]，fork创建子进程
## 网络IO模型
https://zhuanlan.zhihu.com/p/614204046
![](https://pic1.zhimg.com/v2-50b04c875eba45a66a042f4ee4ce2eb8_r.jpg)

I/O多路复用
- [[操作系统#epoll]]
Reactor模式
- [[网络#Reactor模式]]

## 数据结构


基础 -> String、Hash、List、Set、SortedSet（Zset）
高级 -> HyperLogLogs、GEO、Bitmaps、Pub/Sub
![[Pasted image 20240309233952.jpg]]

### 基础数据类型
#### String
![](https://cdn.xiaolincoding.com//mysql/other/516738c4058cdf9109e40a7812ef4239.png)
底层是SDS结构实现的，是为了提高效率而设计的
底层都是*字节数组*形式存储。字符串类型的最大空间不能超过*512M*

#### Key的层级结构
用`:`间隔层级
value为字符串化的*JSON*代码
`set my:user:1 '{"id":1,"name":"Jack","age":21}`
`set my:product:1 '{"id":1,"name":"小米11","price":4999}`

### 集合数据类型
每个集合数据类型都有两种数据结构：
![https://cdn.xiaolincoding.com//mysql/other/9fa26a74965efbf0f56b707a03bb9b7f-20230309232518487.png](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/redis/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84-lastnew.png)
##### 压缩列表
list hash zset
压缩列表是 Redis 为了节约内存而开发，由连续内存块组成的顺序性数据结构
通过zllen可以直接定位头尾节点O(1)，而中间元素需要遍历O(N)
![](https://cdn.xiaolincoding.com//mysql/other/a3b1f6235cf0587115b21312fe60289c.png)

节点中记录了：
1. prevlen 前一个节点占用的字符数
2. encoding 当前节点的编码类型：string / int
3. data 实际数据

###### 连锁更新
因为compact存储的，所以更新/增加时，如果空间需要修改则连锁更新后面的节点，性能很低

##### 整数集合
当set只有整数元素时且元素数量不大时使用
还是用于节约内存空间，底层是compact的

#### Hash
把JSON的键值对变成HashMap
`HSET my:user:1 name "Jack"`
`HGET my:user:1 name` // Jack

#### List
与Java的LinkedList相似
`LPUSH/RPUSH` list左插入/右插入
`LPOP/RPOP`
#### Set
与Java的HashSet相似
`SADD/SREM` 添加/删除

#### SortedSet / Zset
与Java的TreeSet相似
`ZADD/ZREM` 添加/删除



## 过期及删除
### 过期的实现
key设置过期时间时，会将key存储到过期字典中，其中存储key对应的过期时间(long long类型)

查询key时，会先检查过期字典，判断有没有这个key并判断是否过期，没有或没过期则正常访问数据

### 过期删除策略
主动删除：DEL / Unlink
定时删除：设置一个定时事件，时间到了则触发事件，自动删除key
- 保证key尽快删除
- 但需要消耗CPU资源
惰性删除：访问key时删除
- 内存不友好，不删除会一直在内存中
- CPU友好，不需要设定定时事件
定期删除：每隔一段时间随机检查一定数量的key，删除过期的key
- 是定时和惰性的折中方案

## 内存淘汰
配置文件 redis.conf 中，通过参数 `maxmemory <bytes>` 来设定最大运行内存，运行内存达到最大运行内存，则触发内存淘汰策略

### 淘汰策略
分为对所有key和对设置了过期时间的key的两套策略：
1. 不淘汰（3.0后默认）：写新数据会报错
2. TTL：淘汰最早过期的key
3. LRU（3.0前默认）
4. 随机淘汰
5. LFU：最少使用



# 持久化
## RDB
保存快照文件，默认调用save 命令或停止服务时调用主进程阻塞式进行持久化，否则使用bgsave异步持久化
```shell
# 900秒里面有1次修改就执行 bgsave
save 900 1
# 300秒里面有10次修改就执行 bgsave
save 300 10
# 60秒里面有10000次修改就执行 bgsave
save 60 10000
```
bgsave通过fork创建子进程，子进程共享主进程的内存数据，异步存储数据库中的数据，fork采用的是copy-on-write^[当主进程执行读操作时，访问共享内存, 当主进程执行写操作时，则会拷贝一份数据，执行写操作]技术，因此子进程写磁盘的时候不会脏写。
不使用多线程，是因为多线程共享内存，在主线程写RDB时会有脏写的问题。

#### 缺点
1. 执行间隔时间长，两次写入有可能数据丢失
fork子进程、写RDB很耗时

## AOF
追加文件的形式，把每一个**写命令**记录在AOF文件中，可以看作是一个binlog
默认写到缓冲区，每一秒钟写入磁盘中，可以设置1. 直接写进磁盘 2. 只写到缓冲区，操作系统决定啥时候刷新到磁盘中

通过 **bgrewriteaof** 实现aof文件重写，避免记录前面的写命令被后面覆盖，但是aof文件里面还是记录了的问题

# 缓存
## 数据一致性
[高并发场景下，6种方案，保证缓存和数据库的最终一致性！](https://cloud.tencent.com/developer/article/1934375)

> [!info] 延迟双删
> 先删除Redis的缓存，防止更新sql时读取到错误的值
> 再更新sql
> 延迟数百毫秒后再次删除Redis缓存，可能这段时间sql还没更新好

### Cache-Aside 旁路缓存
![|400](https://ask.qcloudimg.com/http-save/1422024/19a8e256f47fa1bb61b8a6cde33a3198.png)
##### 为啥删除缓存，而不是更新缓存？
1. 性能：删比更新快，
2. 安全：多线程更新可能导致不是最新

##### 为啥先更新数据库然后删除缓存？
1. 防止缓存读到旧的数据
2. 如果一定要删除缓存可以使用延迟双删

##### 旁路缓存一定一致吗？
也不一定
###### 情况一 - 读后写
1. 线程1读缓存，没有，去找DB读数据
2. 线程2在线程1访问DB的时候更新了缓存
3. 线程1读到DB数据，写进Cache
4. 现在Cache中的数据错了
![|400](https://ask.qcloudimg.com/http-save/1422024/ca0ddd92b6f2c178359b3174b0ff07bd.png)
###### 情况二 - 写后读
![|400](https://ask.qcloudimg.com/http-save/1422024/232d1e2d8028bd1cbe6a96d719dcfe0e.png)
线程1写，更新DB，还没删Cache的时候，线程2读了Cache，命中了
解决方案：可以对更新DB和删除Cache加锁

### 补偿机制
##### 消息队列
删除失败时  发送需要删除的key到MQ
在MQ中再次重试删除  直到删除成功

需要在代码中配置一个 trigger，代码有侵入性
##### 订阅binlog
MySQL 生成binlog
redis 订阅binlog

##### 读穿、写穿、写回
Read-Through
![|300](https://ask.qcloudimg.com/http-save/1422024/57bf44c3eb2a112ff14ce1ad5b7ded80.png)
Write-Through：多线程可能导致不一致，需要使用事务
![|200](https://ask.qcloudimg.com/http-save/1422024/16372a7c3a3aa76a83b6c5e26e8f9731.png)
Write-Behind：写回
![|400](https://ask.qcloudimg.com/http-save/1422024/7e1396e0550070e38cdb37fd77ab802e.png)




## 缓存击穿、穿透、雪崩
##### 击穿
高频缓存同时失效，请求直接打到sql服务器上
解决方案：
1. **互斥锁**：使用互斥锁保证只有一个请求能够访问数据库并重新加载缓存。
2. **定时刷新**：在缓存项即将过期前，提前进行缓存的刷新操作，避免过期时的大量请求同时到达。
3. 分散缓存过期：过期时间上加一个随机数，降低同时失效的概率
##### 穿透
查询数据库中不存在的数据，由于缓存中没有该数据，请求直接打到sql服务器上
解决方案：
1. **布隆过滤器**：使用布隆过滤器拦截大量不存在的数据请求，避免这些请求到达数据库。
2. **空值缓存**：对于数据库查询返回空值的情况，也将其缓存起来，但设置一个较短的过期时间。
##### 雪崩
和击穿类似，就是高频缓存同时失效或者其他原因缓存宕机了，请求直接打到sql服务器上，造成了一系列的连锁反应，就像雪崩一样
解决措施：
1. 如果是缓存击穿造成的，就使用防止缓存击穿的方法
2. **缓存预热**：在系统启动或缓存数据过期后，提前加载缓存数据，避免在高流量时段重建缓存。
3. **集群和负载均衡**：通过缓存集群和负载均衡技术，分散请求压力，提高系统的可用性和稳定性。    
### 布隆过滤器 bloom filter
通过多个hash函数在一个bitmap上记录某个key的存在情况
具体地，k个hash函数算出k个hash值，对应到bitmap的多个位置，将这多个位置的值置为1
查询时，查看这几个hash值对应位置是不是都是1，如果都是1，则说明key不存在过滤请求

##### 缺点
1. 不支持删除：可能是其他key对应的位置
2. 存在误判：hash冲突，可能存在误判；但一定不会漏判
3. 误报率随着key增加而增加
4. 不能扩容：扩容就要rehash，但我不知道key



## 热key问题

# 分布式锁
## SETNX + EXPIRE
`SET lock_key unique_value NX PX 10000`
value中存一个unique_value，标识这个锁是哪个客户端/线程加的锁，防止释放锁时释放了其他客户端/线程加的锁
redisson使用客户端的UUID和线程id，可以理解为某个客户端的某个线程加锁

## redisson
实现了更高级的可重入的分布式锁
底层用LUA脚本实现原子性，[[网络#Reactor模式|Netty]]实现NIO
还实现了读写锁、公平锁等

#### 加锁原理
redis中存储以下数据：key(锁名)：value(客户端的UUID和线程id，count加锁次数)
```json
myLock:{
"b983c153-7421-469a-addb-44fb92259a1b:1":2
}
```

加锁时，如果锁不存在，则加锁
若已存在，则判断是不是自己的线程，不是则return
若是，则对已加锁线程+1，并设置过期时间，实现可重入

释放时counter-1，并检查counter是否为0，是则发布释放锁的消息并删除key
否则设置设置过期时间

##### 看门狗
后台是一个工作线程，每隔10秒钟的时间会check当前线程是否还持有锁，如果持有锁就将锁的过期时间延长至30s


# 发布订阅
基于频道的发布和订阅
- publisher向指定的channel发送消息，发布的消息不会持久化
- subscriber订阅channel后阻塞，直到收到消息，或者执行订阅or取消订阅操作
- channel向所有订阅的subscriber发布消息

基于模式（pattern）的发布和订阅，通过模式匹配，批量订阅符合模式的频道

#### 底层实现
channel数据结构
![](https://pdai.tech/images/db/redis/db-redis-sub-3.svg)
是map+链表的形式：map中存{频道名:subscriber链表}
发布时，遍历对应channel的subscriber链表，发送消息
订阅时，在map中对应的channel链表中加入subscriber节点

pattern数据结构
![](https://pdai.tech/images/db/redis/db-redis-sub-10.svg)
是链表形式，节点中存储：模式（pattern）和相应的subscriber

# 集群
## 主从架构
对master写，slave读
在slave上执行(或在配置文件中写入)`slaveof <master ip> <master port>`命令，实现主节点的链接，执行命令是临时的，写配置是永久的

### 数据同步的原理
#### 全量同步
创建时将master的RDB发给slave
然后再把记录RDB时的操作写入replog(类似binlog)，发给slave

从节点第再次连接时，通过 repid 和 offset 判断需不需要同步
- repid：master生成的一个唯一id，slave第一次链接时继承这个id
- offset：随着replog增加而增加，master每写replog都会增加，slave每次同步replog也会增加，因此slave的一定小于等于master的offset，如果slave 的 offset小于master的offset则需要更新
#### 增量同步
slave重连时，通过replog获取offset后面的数据
如果slave下线后replog太多，覆盖了之前没有同步的数据，会发生增量同步失败，需要使用全量同步
#### 优化
设置主-从-从架构，从-从节点访问master的从节点，减轻master压力
设置无磁盘复制，RDB文件不写入磁盘，直接通过网络同步给slave

## 哨兵架构
解决master宕机的问题，主从自动切换，当master宕机了哨兵(Sentinel)选一个slave升级为master
作用：监控master和slave的状态，自动故障恢复（选新的master），通知故障信息以及客户端新的master是什么
### 状态监控
心跳包实现（每1s发一个ping，没反馈就认为下线了）
主观下线：1个sentinel认为节点下线了
客观下线：半数以上的sentinel认为节点下线了

### 选举master
排除slave中与master断开时间长的（超过配置的阈值
选优先级高的(配置文件中设置，默认优先级是一致的)、offset大的
最后选slave的id小的
**故障转移**步骤：
1. 让选出来的slave成为主节点，执行`slaveof no one`命令
2. 把新的配置文件发给剩下的slave，告诉他们新的master的ip 和 port
3. 把故障的master也标记为slave
4. 等故障的master恢复了，原master会清空数据，[[#全量同步]]接受新master的RDB
## 集群架构
创建多个master，用master实现哨兵的功能，同时多个master实现数据库的分片
数据存储通过Hash槽实现key的分片存储、访问
可以通过手动故障转移实现master 的切换
### Hash槽
Redis会把每一个master节点映射到 0~16383共16384(16k，14bit)个插槽(hash slot)上
数据key不是与节点绑定，而是与插槽绑定。rediss会根据key的有效部分计算插槽值
key的有效部分：
- key中被`{}`包裹的部分，如果没有`{}`整个key都是有效部分
数据跟Hash槽绑定，而不是节点
节点删除时：把对应节点的Hash槽转移
节点扩容时：把对应节点的Hash槽转移

## 脑裂问题
https://juejin.cn/post/7132768535486398495
#### 什么是脑裂
对于master节点，由于**master和sentinel间的网络波动**或者master CPU利用率太高了，没有及时发心跳包给哨兵，错误地被哨兵认为下线了
此时哨兵集群[[#选举master]]，但是这期间客户端可能继续向旧master写数据，而此时哨兵已经选出来新master了，此时有两个master，称为*脑裂*
此时旧master上线后，被哨兵降为slave，触发[[#全量同步]]，会删除原本的数据，但是**故障期间写入的数据就丢失**了

#### 应对方案
问题是出在原master发生假故障后仍然能接收请求，因此可以限制maste的条件避免脑裂
两个参数
- `min-slaves-to-write`：master可写的最少的salve数量
- `min-slaves-max-lag`：slave给master发ACK的最大延迟

这两个配置项组合后的要求是，主库连接的从库中至少有 N 个从库，和主库进行数据复制时的 ACK 消息延迟不能超过 T 秒，否则，主库就不会再接收客户端的请求了。


# 优化
### 大Key问题
Redis大key问题指的是某个key对应的value占用空间大或集合的元素数过多
会导致Redis的性能下降、主从同步延迟等问题如：
- 客户端阻塞：单线程查询耗时，造成客户端阻塞
- 网络阻塞：每秒访问量大，需要更大的网络带宽
- 阻塞主线程：删除大key时主线程阻塞

##### 大key查找
bigkeys命令：
- 阻塞查找，最好在slave节点上查找
- 只能找到每种类型中最大的key，不能找到前n个keySet
- 只能找到集合数量的多少，而不知道实际的内存大小
SCAN命令：
- 扫描数据库，然后通过LEN函数，计算每个value集合的元素数，再估计内存占用量
MEMORY USAGE命令：
- 查询一个key-value占用的空间

##### 大key的删除
如果直接删除大key，释放大量内存，空闲内存块链表的操作耗时，会阻塞Redis
可以使用分批删除或异步删除
分批删除：每次删除100个元素，多次删除
异步删除：Redis4.0 用 unlink，异步删除key-value对

##### 出现原因
1. 业务设置不合理：key如果可以再细分，可以分到多个key中，而不是存在一起
2. 过期时间设置不当，或者没有设置
3. value的数据量大：如热门帖子的楼层太多




### 热查询/热Key问题
某个Key被频繁访问，导致流量集中，服务器高负载
影响其他key的读写请求
热key打在同一个redis上，没法通过扩容解决
可能redis访问失败造成击穿

#### 找热key
根据经验预测，提前处理
客户端收集：在访问redis前统计
代理层收集：在redis和客户端中加一个代理，
hotkeys命令：底层通过scan 和 object freq实现
抓包分析


#### 解决方案
集群：扩容热key，负载均衡请求压力
二级缓存：对redis加一个缓存，直接在JVM本地中获取
- 要注意防止本地缓存太大，影响性能
- 处理一致性问题
备份：生成多个key，热key加上后缀，分片到多个Redis实例


