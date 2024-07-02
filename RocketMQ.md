## 架构
[[RocketMQ 的存储]]
![](https://s3.51cto.com/oss/202212/31/a4196146814aea4ab9337930f399107efbb673.png)

*Producer* ：生产者，通过 MQ 的负载均衡模块选择相应的 Broker 集群队列进行消息投递，投递的过程支持快速失败并且低延迟。
*Consumer* ：消费者，支持 push 推，pull 拉两种模式对消息进行消费。

*NameServer* ：一个非常简单的 Topic 路由注册中心，类似 zookeeper，支持 Broker 的动态注册与发现。
*BrokerServer* ：Broker 主要负责消息的存储、投递和查询以及服务高可用保证 。


### NameServer
一个简单版的Zookeeper

NameServer每10秒通过线程池扫描Broker，如果检测到宕机就再路由表中删除
路由表的变化不会马上通知Producer（为了减少NameServer的复杂性
NameServer集群间不通讯，即某个时刻数据不一致并不影响消息的发送（追求简单高效

![[Pasted image 20240615002037.png]]


### Broker
#### 主从复制
*Master Broker只接收*producer的消息，同步给从Broker，*Slave Broker发送*消息给consumer

从节点每5秒上报消费的偏移量，然后主节点根据偏移量传消息给从节点，从节点进行存盘

分为同步和异步复制：
同步复制在主节点刷盘再调用复制操作，等待从节点返回确认后返回ACK
异步复制：调用复制操作后直接返回ACK，异步等待从节点的确认


### Producer
##### 发送流程
1. 消息校验
2. 在本地缓存中找有没有topic的路由信息，没有的话就请求NameServer，并缓存
3. 选择消息队列，轮询负载均衡地选MessageQueue
4. 发送消息，设置消息的唯一ID，对消息进行压缩（可选的），然后用requset方法发送消息




### Consumer

pull时一次性取32条消息，存在本地的Queue中


## 消息类型
### 延时消息
[[RocketMQ 延时消息]]
使用场景：
1. 下单后，指定时间内未支付，取消订单

消息发送后，**会在Broker中进行存储**，等过了设定的延迟时间之后，消息才会被 consumer 消费，通过`setDelayTimeLevel`设置延时级别并指定为延时消息，共有18个级别，从1s到2h
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05d9d6ab51e940a791d8e93ebb4e8bbd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 底层实现
设置专用的消息队列存储延时消息，通过Timer、**TimerTask**定时转发到普通消息队列


具体流程：
1. Broker 收到 message 送入专门用于延时的18个topic和queueId中，对应着18个延时级别，并通过设置property来保存原本的topic和queueId
2. 然后Broker中每一个queue，都通过Timer和TimerTask执行定时任务，处理对应的消息
3. 到时后把这个消息根据保存的topic和queueId转发到普通的消息队列，正常消费


#### 自定义延时
java可以用TimerTask、延迟线程实现延时，但是其原理都是基于最小堆实现的`延迟队列DelayQueue`，插入任务的时间复杂度为 Olog(n)，消息 TPS 较高时性能仍不够快
Kafka使用[[操作系统#TimeWheel 时间轮|TimeWheel 时间轮]]  实现O(1)优化延时操作

##### CommitLog保存超长延迟数据
由于CommitLog只能保存7天的消息，所以延迟消息的log以链表的形式写在单独的文件中
并在消息投递后删除


### 事务消息
分布式事务消息，在普通消息基础上，支持*二阶段提交*。将二阶段提交和本地事务绑定，实现全局提交结果的一致性。
不能保证消息消费结果和上游事务的一致性，只保证**最终一致性**

![](https://rocketmq.apache.org/zh/assets/images/transflow-0b07236d124ddb814aeaf5f6b5f3f72c.png)

Producer发送半消息给Broker，收到ACK后执行本地事务，然后执行commit或者rollback
如果commit，就发送消息给Consumer，否则删除消息

如果断网或Producer重启，Broker长时间没收到二次确认，就进行回查，如果超过最大回查时间则强制rollback





### 顺序消息

### 广播消息



## 消息存储
BrokerServer 使用[[#内存映射]]存储消息


异步刷盘（默认）：只要Broker写入了PageCache就**返回ack**，后台线程异步刷盘PageCache
同步刷盘：在执行完刷盘操作才返回ack
决定刷盘策略的参数：FlushDiskType：SYNC_FLUSH（同步刷盘

### 存储与发送
存储使用顺序写

使用[[网络#零拷贝]]提高读取->发送的速度
- 由于使用了mmap的方式，其一次只能映射1.5-2G的文件，所有默认单个CommitLog日志数据文件为1G

### 存储结构
![[Pasted image 20240614231410.png]]
#### CommitLog
存储了消息相关的所有数据（包括：Topic，QueueID，Message
由于[[网络#零拷贝]]的限制，CommitLog的大小限制为1G，存满后，新建一个文件

所有的消息都存在CommitLog中，为了快速找到消息，加入了索引`ConsumerQueue`

#### ConsumerQueue
是CommitLog中数据的*索引*，Consumer消费时直接问ConsumerQueue找消息的存储位置，然后根据索引读消息

![](https://s6.51cto.com/oss/202212/31/d6ca5a25224f4a8b86066534fa7547ab3b0094.png)

其中记录了某个消息在CommitLog中的位置、大小和Tag的哈希值
其中维护了三个指针：minOffset，consumerOffset，maxOffset

#### IndexFile
索引文件，其提供了一种通过*key或时间区间*来查询消息的方法
这种通过IndexFile来查找消息的方法不影响发送与消费消息的主流程
![](https://s2.51cto.com/oss/202212/31/57535a990bbf8729ab9237634a81207a69a04d.png)

类似HashMap的结构，使用拉链法解决hash冲突












### 网络通信
使用扩展的多线程[[网络#Reactor模式|Reactor]]
Reactor监听到连接，创建SocketChannel，多路复用监听数据，拿到数据后
用3个线程组执行业务
1. defaultEventExecutorGroup执行验证、编解码、检查等预处理
2. Worker根据消息的元数据，封装成对应的task任务，提交给对应的执行线程
3. 对应的执行线程，包括：发送消息的sendMessageExecutor等



### 内存映射
即[[网络#零拷贝]]中的mmap
为了提高I/O性能，存储的文件都使用内存映射存储，所以文件大小都一样，



## 消息重试
不支持**广播模式**

集群模式消费失败的情况下，可以返回`RECONSUME_LATER`，消息就会之后重试
重试默认会进行重试 16 次，重试次数越多，与上次重试的时间间隔就加长一点，底层是基于[[#延时消息]]实现的
重试16次还没成功 -> 加入[[#死信队列]]

**顺序消息**失败会间隔1s，不断重试，会阻塞消息队列


### 死信队列
保存3天，之后会删除
死信队列对应一个GroupId，而不是单个Consumer实例











## 负载均衡
https://www.cnblogs.com/hzzjj/p/16552514.html

Producer发送消息使用轮询MessageQueue的方式实现负载均衡，而MessageQueue可以部署在同一台或不同机器上。

不使用广播模式时：Consumer组通过均摊地订阅MessageQueue实现负载均衡
广播模式要求Consumer消费所有的Broker

#### Rebalance机制
当Consumer/Broker缩扩容时，重新分配订阅关系
可以提高并行性，但是rebalance期间存在一些问题：
消费暂停、消费重复、消费突刺/堆积

#### 其他负载均衡算法
[[分布式#负载均衡]]





# 问题

### 消息堆积和延迟
大消息、I/O、下游任务、事务消息处理慢

1. 查什么原因，I/O或下游任务导致的变慢，优化代码
2. 临时扩容，增加消费端的数量、扩大队列长度
3. 设置优先级，降级一些非核心的业务、对生产者限流
4. 负载均衡

### 消息丢失
通过补偿机制重试
##### 生产/消费时丢失
消费者在处理完业务后再发送ack
生产者只要能正常收到 Broker 的 ack 确认响应，就表示发送成功

##### Broker宕机
写磁盘需要通过cache，不是立刻就刷盘的，再未刷盘时宕机会导致丢失

解决方法：
1. 同步刷盘：写一条刷一条
2. 使用集群部署，至少同步到两个节点后，才返回ack

##### 再检查
producer对每个消息都指定一个全局唯一id，consumer通过filter校验

### 消息重复/幂等
消息丢失重试的过程可能产生重复的消息，即：如何解决消费端幂等性问题

1. 发送时重复：网络波动or宕机，Broker持久化了，但是确认Producer没收到，再次发送时重复
2. 投递时重复：Consumer处理了，但是某些原因Broker没收到确认，重试时重复了
3. [[#Rebalance机制]]时重复：扩容时Consumer可能收到重复消息

使用业务唯一标识判断是否已经消费：UUID、全局唯一ID、版本号等

### 顺序性
生产顺序性：相同消息组的消息按照先后顺序被存储在**同一个队列**
如需保证消息生产的顺序性，则必须满足以下条件：
1. 同一消息组（MessageGroup）：只有同一消息组内的消息可以保证顺序性
2. 单一生产者：仅支持单一生产者，不同生产者分布在不同的系统，即使设置相同的消息组，也无法判定其先后顺序。
3. 串行发送：如果生产者使用多线程并行发送，则不同线程间产生的消息将无法判定其先后顺序。

消费顺序性：通过消费者和服务端的协议保障消息消费严格按照存储的先后顺序来处理。
如果是PushConsumer可以保证一条一条推送，如果是普通的consumer则需要业务自己实现

消费失败导致重试，会阻塞后面的消息，如果超过重试次数，会被丢弃，可能导致乱序




## 面试题
版本5.0和4.X的区别？




# Kafka
