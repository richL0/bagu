DML (Data Manipulation Language) 用于操作数据库中的数据，如插入、更新、删除和查询*数据*。
DDL (Data Definition Language) 用于*定义、修改和删除*数据库对象的*结构*，如创建表、索引、视图等。
DQL (Data Query Language) 用于*查询数据*库中的数据，主要是指SELECT语句。
# 基础

### 三大范式

| 范式  | 规则    | 说明           |
| --- | ----- | ------------ |
| 1NF | 列原子性  | 无重复列，避免数据冗余。 |
| 2NF | 全依赖   | 主键唯一确定所有列。   |
| 3NF | 非传递依赖 | 非主属性直接依赖主键。  |
- **1NF**
	- 确保表中每一列都是原子的，不可再分。
	- 地址里包含省、市，不满足需求，不符合第一范式。应该把省、市、地址 分开
- **2NF**：
	- 表中的非主键字段完全依赖于主键，消除部分依赖。
	- 订单表中，`订单号`是主键，`客户姓名`和`客户地址`完全依赖于`订单号`。应该两张表 一个是`订单表` 订单对应有`客户ID`, 另一张表是`客户信息表`
- **3NF**：
	- 消除非主键字段之间的依赖关系，即非主键字段不依赖于其他非主键字段。
	- 如果`客户姓名`依赖于`客户ID`，而`客户ID`又是订单的外键，这违反了3NF，因为`客户姓名`间接依赖于`订单号`。正确的做法是将客户信息单独成表，订单表只保存`客户ID`。

### 幂等
重复调用的结果和单次调用的结果一致

| 操作  | 幂等  | 实例                                                                                                               |
| --- | --- | ---------------------------------------------------------------------------------------------------------------- |
| 查询  | ✔   | 不对数据产生变化                                                                                                         |
| 新增  | ✔/❌ | 插入唯一主键，具备**幂等**性。<br>不是主键，不具备幂等性。                                                                                |
| 修改  | ✔/❌ | update user set `point = 20` where userid=1 具备幂等性。<br>update user set `point = point + 20` where userid=1 不具备幂等性 |
| 删除  | ✔   | delete from user where userid=1                                                                                  |
#### 幂等方案

| 方法   | 说明                                                                  | 优缺点                                                                                      |
| ---- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 唯一索引 | 利用数据库的唯一索引约束，解决了在insert场景时幂等问题。业务生成全局唯一的值。 通常使用唯一主键索引。              | 优点：实现很简单                                                                                 |
| 乐观锁  | 乐观锁 [[并发与多线程(JUC)#CAS]]                                             | UPDATE user SET point = point + 20, version = version + 1 `WHERE userid=1 AND version=1` |
| 分布式锁 | 使用token + Redis                                                     | 缺点：业务请求每次请求，都会有额外的请求                                                                     |
| 去重表  | 把唯一主键插入去重表，再进行业务操作，且他们在同一个事务中。这个保证了重复请求时，因为去重表有唯一约束，导致请求失败，避免了幂等问题。 |                                                                                          |

### 外键
##### 优点
1. 保证数据的完整性和一致性
2. 级联操作方便
3. 将数据完整性判断托付给了数据库完成，减少了程序的代码量
4. 如果通过物理外键解决bug比逻辑外键更简单
##### 缺点
1. 性能问题：每次向A表中插入数据都会去B表查询是否有对应数据。如果不止一个外键或者批量插入/更新，性能会很差
2. 并发问题：在使用外键的情况下，每次修改数据都需要去另外一个表检查数据，需要获取额外的锁。若是在高并发大流量事务场景，使用外键更容易造成死锁。
3. 扩展性问题：比如表结构重构，迁移(mysql迁移到oracle)，分表分库
![[Pasted image 20240306171847.png]]


### 安全
#### 防止SQL注入

- **权限区分**：普通用户与系统管理员*权限严格区分*，限制执行敏感操作。
- **ORM框架**：推荐使用*MyBatis-Plus*等，有效防止SQL注入。
- **参数化语句**：用户输入通过*参数传递*，避免直接嵌入SQL语句，减少注入风险。 即用户的输入绝对不能够直接被嵌入到SQL语句中
- **输入验证**：对用户输入进行类型、长度、格式和范围的*严格验证*，过滤恶意代码。
- **数据库安全参数**：利用SQL Server的Parameters集合等*数据库自带的安全参数*，进行类型检查和长度验证。
- **多层环境防护**：在*多层应用环境*中进行输入验证，确保数据在进入可信区域前通过验证。
- **陷阱账号**：设置*假管理员账号*，吸引攻击者，保护真实账号安全。
- **漏洞扫描工具**：定期使用专业工具检查系统安全性，发现潜在攻击点。

# 优化
## 设计优化
### 表设计优化
InnoDB存储引擎使用文件来管理表的数据和索引，每个表文件大小限制为64TB，页大小为16KB
为了更好的性能，每个表的数据超过500w就应该考虑分表了，最好不要超过2000w行
### 插入优化
1. 批量插入
2. 手动提交事务，避免频繁创建事务
3. 主键顺序插入
4. 对于大批量数据的插入，使用`load`指令

### 主键优化
数据组织方式
在InnoDB存储引擎中，表数据都是**根据主键顺序存放**的，这种存储方式的表称为*索引组织表*(index organized table IOT)
-  页分裂
主键插入时遇到页满了，则分裂页
把要插入的页的一半移到一个新页
-  页合并
删除时当前数据不足50% 则查看前后的页可不可以合并
#### 设计原则
1. 降低主键长度
2. 插入时按顺序插入
3. 不要使用UUID 或 身份证号做主键
	1. UUID / ID无序
	2. 长度太长
4. 避免修改主键


## 查询优化
### 优化
#### 索引优化
1. 需要对`Order By`的字段创建索引
2. *联合索引*
	1. 避免破坏*最左前缀*
	2. 前后的排序方向要一致 (`asc` `desc`)
	3. 排序方向不一致  可以创建不同的索引 `create index idx_user_age_phone_ad on tb_user(age asc ,phone desc);`
3. 可不避免的`filesort`，可以扩大`sort_buffer_size` 的大小

#### limit优化
对于大数据量的`limit`查询需要一步一步往后查
如`limit 2000000,10`,需要MySQL排序前`2000010`记录，仅仅返回`2000000-2000010`的记录，其他记录丢弃，查询排序的代价非常大。

1. 根据id跨过offset
```sql
SELECT * FROM  product_new WHERE id>300396 ORDER BY id LIMIT 10  -- 根据id查询，并且使用where过滤
```
2. 先对索引覆盖索引查到id，然后对用id去查数据
```sql
SELECT * FROM product_new INNER JOIN 
(SELECT id FROM  product_new ORDER BY id LIMIT 300000,10 ) a 
ON product_new.id=a.id
```

#### count优化
`count()`返回累计值 参数不是`NULL`则累计值+1

MyISAM引擎把一个表的总行数存在了磁盘上，因此执行`Cout(*)`的时候会直接返回这个数，效率很高

- `count(*)`：InnoDB遍历表不取值  服务层按行+1
- `count(1)`：InnoDB遍历表不取值  服务层填入 '1' 按行+1
- `count(主键)`：InnoDB把每个主键取出来  服务层按行+1
- `count(字段)`：有`not Null` 约束: InnoDB把每个主键取出来  服务层直接+1，没有`not Null` 约束: InnoDB把每个主键取出来  传给服务层  服务层判断是否+1
**耗时**：`count(*)` = `count(1)` <  `count(主键)` < `count(字段)`

#### update优化
InnoDB有行锁
- 如果 `where` 字段*没有索引*或者*索引失效*  会**锁整张表**
- 主键默认有索引 尽量根据主键更新



## 慢查询
#### 定位
##### 慢查询日志
```mysql 
SET GLOBAL slow_query_log = on; -- 开启慢查询log
SET GLOBAL long_query_time = 1; -- 慢查询阈值单位：秒
SET GLOBAL slow_query_log_file="slow.log"; -- 日志路径
```
日志中包括*查询语句，查询时间*等信息


##### `EXPLAIN`性能分析
```mysql
explain select * from teacher_info;
```

| 字段          | 作用                   |
| ----------- | -------------------- |
| select_type | 查询类型                 |
| **type**    | 查找方式                 |
| **key**     | 实际用到的索引              |
| key_len     | 实际用到的索引长度            |
| ref         | 本次查询引用了哪些字段，哪些数据进行查找 |
| rows        | 完成当前查询，预计所要读取的行数     |
| **Extra**   | 额外的信息                |
###### type字段
返回查找方式，执行计划中重中之重的参数，整个SQL优化就是围绕此参数进行优化
- system：系统表的查询，仅返回一行结果，速度最快
- const：常量查询，基于常量条件查询的，用于*主键或唯一索引*的查询
- eq_ref：唯一索引访问，通过唯一索引查找。这种类型的查询通常用于使用主键或唯一索引进行关联查询，每个索引值只有一条匹配的结果
- ref：非唯一索引访问，通过*非唯一索引*查找
- range：范围扫描，对*索引使用了范围查找*
- index：索引扫描，MySQL 使用*非唯一索引扫描*
- ALL：全表扫描，性能较差
###### extra字段
表示查询后是否还要进行额外的操作再生成结果集
- Using index: 使用了覆盖索引，数据可以直接从索引获取，无需访问表
- Using where: 使用了 WHERE 条件过滤
- Using temporary: 需要创建临时表处理结果集，通常发生在需要进行排序、分组或多表连接的情况下
- Using filesort: 需要进行额外排序，未使用索引来排序
- Using index condition: 使用了索引条件进行过滤
- Using join buffer: 使用了连接缓冲区
- Distinct: 使用了 DISTINCT 关键字进行去重。
- Full scan on NULL key: 执行了全索引扫描，但索引键值为空
- Range checked for each record: 对每条记录都进行了范围检查。
- Using index for group-by: 使用索引进行分组
- Using index for order by: 使用索引进行排序




# 事务
事务是一组操作的集合，它是一个*不可分割*的工作单位，事务会把所有的操作作为一个整体一起向系统提交或撤销操作请求，即这些操作*要么同时成功*，*要么同时失败*。
## 隔离级别
事务隔离通过[[#MVCC]]机制实现了*不加锁的线程安全*

*脏读*：读取到未提交事务所做的修改。
*不可重复读*：在同一个事务中，多次读取同一数据时，结果可能不同。


*幻读*：两次相同的查询操作，读到的数据的行数不同。即，在同一个事务中，执行相同的查询，查不到记录，插入时又提示已经有这条记录了。

[innoDB幻读](https://juejin.cn/post/7134186501306318856)通过MVCC避免了快照读的幻读，但是不能避免当前读的幻读


| **隔离级别** | 脏读  | 不可重复读 | 幻读  | 指令                 |
| -------- | --- | ----- | --- | ------------------ |
| **未提交读** | ✔   | ✔     | ✔   | `Read Umcommitted` |
| **提交读**  | ❌   | ✔     | ✔   | `Read Commmitted`  |
| **可重复读** | ❌   | ❌     | ✔   | `Repeated Read`    |
| **序列化**  | ❌   | ❌     | ❌   | `Serializable`     |
## 四个特性  (ACID属性)
#### 原子性（Atomicity）
- 事务要么完全执行，要么完全不执行
- 回滚机制确保数据一致性
- 例子：银行转账
#### 一致性（Consistency）
- 数据库状态始终保持一致
- 遵守所有预定义的约束和规则
- 例子：库存更新
#### 隔离性（Isolation）
- 事务之间相互独立
- 防止并发事务的干扰
- 隔离级别：
  - `读未提交`（Read Uncommitted）
  - `读已提交`（Read Committed）
  - `可重复读`（Repeatable Read）
  - `串行化`（Serializable）
#### 持久性（Durability）
- 事务一旦提交，其效果永久保存
- 数据不会因为系统故障而丢失
- 持久化存储：写入磁盘


## 基础
### 状态
![[Pasted image 20240306203642.png]]
1. 活动（Active）
    - 事务开始执行，但尚未完成。
    - 操作在内存中进行，未影响数据库持久状态。
2. 部分提交（Partially Committed）
    - 事务操作已执行完毕。
    - 更改尚未写入磁盘*尚未写入磁盘*，可能在故障时丢失。
3. 失败（Failed）
    - 事务执行过程中遇到错误。
    - 可能由于数据库错误、操作系统问题、硬件故障或用户中断。
4. 中止（Aborted）
    - 事务失败后*执行回滚操作*。
    - 数据库*恢复到事务开始前的状态*。
5. 提交（Committed）
    - 事务更改被写入磁盘并*持久化*。
    - 事务成功，更改永久性地影响数据库。
### 语法
查看事务提交方式  `SELECT @@autocommit;`
取消自动提交  `SET @@autocommit = 0;`

`BEGIN;` 或 `START TRANSACTION;` 开启事务
`COMMIT;` 提交事务
`ROLLBACK;` 回滚事务
# 索引
索引用于提高*查询*性能
## 结构
#### B+树索引
在innoDB和MyISAM引擎中都支持，适用于**范围查询和排序**

#### 哈希索引
适用于**等值查找**，如=、<>、IN等操作，不适用于模糊查询。
只在*Memeory引擎*显式的支持

#### R-Tree
用于地理数据存储。MyISAM存储引擎中支持

#### 全文索引
用于查找文本中的关键词，全文索引更类似于搜索引擎做的事情，查找条件使用 MATCH AGAINST
全文索引使用*倒排索引*实现，它记录着关键词到其所在文档的映射。

MyISAM存储引擎支持全文索引
InnoDB 存储引擎在 MySQL 5.6.4 版本中也开始支持全文索引

### B树 、 B+树
#### B树
![[Pasted image 20240308152057.png]]
#### B+树
![[Pasted image 20240308152122.png]]
- 叶子节点形成一个单向链表。
- 所有数据都会出现在叶子节点。
#### MySQL B+Tree 索引
![[Pasted image 20240308152457.png]]

### 为什么用**B+树索引**？
1. 相比于二叉树，层级更少，搜索效率高
2. 相比于B树，其叶子节点和非叶子Node 都要存数据，数据占用大量空间，减少指针空间，从而增加了高度
3. 叶子节点有双向链表支持范围搜索和排序

### 二级索引
聚集索引：数据与索引一起存储，叶子节点保存行数据，有且只有一个。
二级索引：数据与索引分开存储，叶子节点保存主键，可以存在多个。

>[!info] 聚集索引选取规则
>存在主键，*主键*索引就是聚集索引。
>不存在主键，将使用*第一个唯一*(UNIQUE)索引作为聚集索引。
>如果表没有主键，或没有合适的唯一索引，则*InnoDB*:会自动生成一个`rowid`作为隐藏的聚集索引。

对于`select * from user where name='Arm';`
二级索引查找`Arm`  得到对应的`id`然后通过 *回表查询*   找到对应的行
![[Pasted image 20240308155450.png|400]]


> [!info] 避免回表的方法：覆盖索引
> 需要查询的字段时存储在索引中的
> 即：创建索引时包括了这个字段
> 避免了回表



## 索引失效
不服从*最左前缀*原则：查询从索引的最左列开始，并且不跳过索引中的列。
- 如果*跳跃某一列*，索引将部分失效（后面的字段索引失效）。

对索引列进行运算或使用函数，索引是有顺序的，如果对其做函数操作会破环顺序
- `SELECT * FROM t WHERE id*3=3000`
- `SELECT * FROM t WHERE ABS(id)=3000

索引出现隐式类型转化：`varchar`不加单引号的话可能会自动转换为`int`型。 
- 索引失效：`SELECT * FROM tb1 WHERE name = 123`  `varchar`自动转换为`int`型，相当于`SELECT * FROM tb1 WHERE CAST(name AS int) = 123` 相当于加了函数运算
- 正确写法：`SELECT * FROM tb1 WHERE name = ‘123’

模糊匹配：索引是按字典序排序的，如果只匹配后面就没法走索引了
- `LIKE` 以`%`或者`_`开头 (*头部模糊匹配*)

`OR` 连接但前后没有同时使用索引

### 索引下推
https://www.cnblogs.com/three-fighter/p/15246577.html
索引下推(Index Condition Pushdown，简称ICP)，它能减少回表查询次数，提高查询效率。
`索引下推`的下推其实就是指将部分上层（服务层）负责的事情，交给了下层（引擎层）去处理。

我们来具体看一下，在没有使用ICP的情况下，MySQL的查询：
- 存储引擎读取索引记录；
- 根据索引中的主键值，定位并读取完整的行记录；
- 存储引擎把记录交给`Server`层去检测该记录是否满足`WHERE`条件。
- EXPLAIN Extra字段:'Using where'

使用ICP的情况下，查询过程：
- 存储引擎读取索引记录（不是完整的行记录）；
- 判断`WHERE`条件部分能否用索引中的列来做检查，条件不满足，则处理下一行索引记录；
- 条件满足，使用索引中的主键去定位并读取完整的行记录（就是所谓的回表）；
- 存储引擎把记录交给`Server`层，`Server`层检测该记录是否满足`WHERE`条件的其余部分。
- EXPLAIN Extra字段:'Using index condition'


## 设计原则
主要用于数据库和文件系统中。
- 对**查询频率高**的字段创建索引
	- 修改频率远远大于查询频率不建议创建索引
	- 修改性能和检索性能是互相矛盾的。
- 为需要经常**条件`where`、排序`oreder by`、分组`group by`和联合`Unioin`操作**的字段建立索引
	- 排序操作会浪费很多时间。
	- 建立索引，可以有效地避免排序操作。
- 尽量**使用唯一索引**
	- 唯一值少的列不适合建立索引。
		- 例如，学生表中学号是具有唯一性的字段。为该字段建立唯一性索引可以很快的确定某个学生的信息。若使用姓名的话，可能存在同名现象，从而降低查询速度。
	- *主键*索引和*唯一*键索引，效率最高。
- 使用**短索引**
	- 索引值长，查询速度会受影响。
	- CHAR(100)类型的字段进行全文检索比CHAR(10)字段慢
- 使用**前缀索引**
	- 全文检索浪费时间
	- 只检索字段前缀，加快检索。
- 索引的**数目不要太多**
	1. 索引占用物理空间
	2. 索引影响性能
		- 降低**增删改**操作效率
	3. 增加优化时间

- 使用联合索引而不是单列索引



# 集群
## 主从架构
https://www.bilibili.com/video/BV1eV4y1D7fs

#### 主从复制
主从复制是指将主数据库的*DDL和DML*操作通过二进制日志传到从库服务器中，然后在从库上对这些日志重新执行（也叫重做），从而使得从库和主库的数据保持同步。

主要涉及三个线程: binlog 线程、I/O 线程和 SQL 线程。
- binlog 线程：负责将主服务器上的数据更改写入二进制日志中。
- I/O 线程：负责从主服务器上读取二进制日志，并写入从服务器的中继日志中。
- SQL 线程：负责读取中继日志并重放其中的 SQL 语句。
![[Pasted image 20240307225806.png]]

#### 数据一致性
主从执行相同的sql可能结果不一致
```mysql
UPDATE t SET updatetime = NOW() WHERE condition;
```
`NOW()` 导致主从数据不一致
可以把Binlog改成Row格式或者Mixedlevel

##### Binlog三种格式
https://www.cnblogs.com/baizhanshi/p/10512399.html
三种格式分别为
1. Statement（默认）：只记录执行语句，不记录每行的变化
2. Row：记录数据的修改情况，仅记录数据被修改成什么了。修改多条记录，会造成binlog很大
3. Mixedlevel：混合前面两个，简单的修改语句用Statement，复杂的语句（从节点不一定正确执行的语句）用Row存

#### 主从延迟
https://www.cnblogs.com/ivictor/p/17331981.html
##### IO线程延迟
- 网络IO瓶颈：压缩binlog（开启`slave_compressed_protocol`
- 磁盘IO瓶颈：关闭从库的[[#双1设置]]，或关闭binlog

##### SQL 线程延迟
- 主库写入量大，从库单线程重放：开启并行复制
- 表上无索引，Row格式，从库每条记录的操作都会进行一次全表扫描，很慢
优化：
1. 从库临时建一个索引
2. 修改从节点扫描算法，（修改参数`slave_rows_search_algorithms` 
- 大事务：分治大事务，每次小批量执行
- 从库有查询操作：1. 消耗系统资源，2. 锁等待，阻塞DDL语句

##### 解决延迟问题
使用缓存，查数据时先查缓存
直接读主节点
数据冗余，

### 读写分离
主服务器处理**写操作**以及高实时性读操作，而从服务器处理读操作。
读写分离能提高性能的原因在于:
- 主从服务器负责各自的读和写，极大程度缓解了锁的争用；
- 从服务器可以使用 **MyISAM，提升查询性能**以及节约系统开销；
- 增加**冗余**，提高可用性。

读写分离常用代理方式来实现，**代理服务器**接收应用层传来的读写请求，然后决定转发到哪个服务器。

## 分库分表
单数据库数据过多时:
1. IO瓶颈
2. CPU瓶颈

- **垂直分库**：不同业务表拆分到不同数据库中。
- **水平分库**：同一业务表的数据按策略拆分到多个数据库中。


**垂直拆表**：按照表的列进行拆分，每个分片包含原表的一部分列。
**水平拆表**：按照表的行进行拆分，每个分片包含原表的一部分行。
### 分片策略
哈希取模: `hash(key) % NUM_DB`
范围: 可以是 *ID 范围*也可以是*时间范围*
映射表: 使用单独的一个*数据库*存储*映射关系*
### 水平分片规则
1. 范围分片
2. 取模分片
3. 一致性hash算法
4. 枚举分片
	1. 按省份分
	2. 按性别分
5. 按日期(天/月)分片
### 分布式事务

### 唯一主键生成
- 数据库自增ID
	- 优点：最简单。 缺点：单点风险、单机性能瓶颈。 
	- 因为ID生成完全依赖于单个数据库实例。如果该实例发生故障或无法访问，整个系统将无法获取新的ID，导致服务中断。
- GUID 、Random算法
	- 优点：简单。 缺点：生成ID较长，有重复几率。
- 为每个分片指定一个 ID 范围
	-  **flickr** 方案
		- 优点：高可用、ID较简洁。 缺点：需要单独的数据库集群。
		- 三个服务器  uid一次自增3  
			- Server0 生成模3余0的uid
			- Server1 生成模3余1的uid
			- Server2 生成模3余2的uid
- 分布式 `ID 生成器` 
	- Snowflake(雪花) 算法
		- 优点：高性能、高可用、易拓展。 缺点：需要独立的集群以及ZK。
- 美团方案
	- 时间戳+用户标识码+随机数
		- 方便、成本低。
		- 基本无重复的可能。
		- 自带分库规则，这里的*用户标识码即为用户ID的后四位*，在查询的场景下，只需要订单号就可以匹配到相应的库表而无需用户ID，只取四位是希望订单号尽可能的短一些，并且评估下来四位已经足够。
		- 可排序，因为时间戳在最前面。


# MVCC 
https://www.bilibili.com/video/BV1Kr4y1i7ru?p=143
多版本并发控制机制(Multi-Version Concurrency Control) 
用于在多个并发事务同时读写数据库时保持数据的一致性和隔离性。(***不加锁保证隔离性***)

> [!info] 读操作
> 1. 默认使用快照读，读取事务开始时的数据，所以不会读其他事务还没提交的数据。
> 2. 选择不晚于其事务开始时间的最新版本，确保事务只读取在它开始之前已经存在的数据。

> [!info] 写操作 （INSERT、UPDATE、DELETE）
> 一个事务执行写操作时，它会生成一个新的数据版本
> 新版本的数据会带有当前事务的版本号，原始版本的数据仍然存在

> [!info] 版本的回收
> 为了防止数据库中的版本无限增长，MVCC 会定期进行版本的回收。回收机制会删除已经不再需要的旧版本数据，从而释放空间。

### undo log
一个回滚版本链
trx_id : 事务id
roll_pointer：指向能回滚的上一条 版本 log
![[Pasted image 20240424222006.png]]


### ReadView
在第一次读取时生成一个ReadView的数组， 保存当前尚未提交的事务id的数组
ReadView，有四个比较重要的东西：[链接](https://juejin.cn/post/7029981510262489102  )
- m_ids：此时有哪些事务在MySQL里执行还没提交
- min_trx_id：m_ids（活跃事务的id）的最小的值；
- max_trx_id：mysql 下一个要生成的事务id，就是最大事务id+1
- creator_trx_id：创建这个ReadView的事务id

#### 访问规则
1. creator_trx_id == trx_id：可以访问，是当前事务更改的
2. trx_id < min_trx_id：可以访问，是已经提交的事务
3. trx_id >= max_trx_id：不可访问，当前事务是ReadView生成后开启的
4. min_trx_id < trx_id < max_trx_id，但是不在m_ids中：可以访问，事务已经提交了

不同隔离级别生成ReadView的时机不同，
- 读已提交（Read Commit）：开启事务后每次执行快照读都生成一个ReadView
- 可重复读（Repeatable Read, RR）：事务第一次执行快照读是生成

## 锁
全局锁
- 对整个数据库上锁
- 只能DQL 不能 DML/DDL
- 用于整个数据库的数据库的备份`mysqldump`
### 表锁
MyISAM, InnoDB都有
分为共享锁（读锁）和独占锁（写锁）
```mysql
-- 表级共享锁，读锁；
lock tables tablename read;

-- 表级独占锁，写锁；
lock tables tablename write;

-- 解锁
UNLOCK TABLES tablename;
```

#### 元数据锁
锁的是表的结构（即表的元数据），避免DML/DQL和DDL的冲突，由系统自动控制
DML/DQL不会改变表结构，加元数据读锁
DDL(对表进行结构变更)时加写锁

元数据锁在事务结束后释放
#### 意向锁
是一种表锁，在InnoDB 中避免行锁和表锁的冲突，在加行锁时对表加*意向锁*
两种类型：
1. 意向共享锁(IS)  与表锁读锁兼容，写锁互斥
2. 意向排他锁(IX)  与表锁的读/写锁互斥

意向锁间不互斥，可以理解为有意向加锁，但没有实际加锁


### 行锁
InnoDB支持，MyISAM不支持行锁
检索唯一索引时加行锁，事务结束（提交或回滚）后释放
> [!warning] 
> 行锁是通过索引对*索引加锁*的，而不是对记录加锁，如果不通过索引检索则加*表锁*

##### 共享锁S
兼容共享锁，因此多个事务允许同时持有一行的读锁
不兼容排他锁，加了共享锁就不能加排他锁

select时会默认加共享锁，也可以显式的
``` sql
SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE;

-- MySQL8.0后
SELECT * FROM table_name WHERE ... FOR SHARE;
```

##### 排他锁X
不兼容其他行锁，会阻塞其他事务对该行的锁（读/写）的获取

insert, delete, update等会自动加排他锁，select需要显式加锁
``` sql
SELECT ... FOR UPDATE
```


#### 分类
##### 记录锁 Record Lock
对索引加锁，如果没有索引就对表加锁
特性
1. 通过索引实现行锁，而不是锁记录。因此，当操作的两条不同记录拥有相同的索引时，也会因为行锁被锁而阻塞。
如：表中a有索引
对 a=1 and b=1 这一行加锁，会通过a的索引加锁
另一个线程对 a=1 and b=5 这行数据加锁会失败
因为第一个线程对 a=1 的数据都加锁了
2. 使用主键索引时，锁定主键索引；使用非主键索引时，会先锁住非主键索引，再锁定主键索引

##### 间隙锁 Gap Lock
对记录间的间隙加锁，解决RR下的幻读问题
间隙锁之间是兼容的，因为间隙锁只限制不能插入数据

##### 临键锁 Next-Key Lock
在RR隔离级别下，默认使用临键锁
临键锁，会锁定当前记录的范围，以及这个记录之前的间隙

当查询的索引是唯一索引或主键索引是，会优化为record lock，不锁定范围
当索引为辅助索引，使用临键锁，会锁定当前记录的范围，还会锁定到下一个键的间隙


# InnoDB

| 特性     | InnoDB         | MyISAM            |
| ------ | -------------- | ----------------- |
| 存储格式   | 行格式            | 固定长度字段和可变长度字段分开存储 |
| 事务     | 支持             | 不支持               |
| 外键     | 支持             | 不支持               |
| 崩溃恢复能力 | 强              | 弱                 |
| 锁机制    | 行锁和表锁          | 表锁                |
| 性能     | 高并发下较好         | 读多写少时较好           |

可以使用[[#行锁]]实现当前读，当前读可能会导致[[#隔离级别|幻读]]

```sql
-- 当前读的一些例子
select * from table where ? lock in share mode;
select * from table where ? for update;
insert;
update ;
delete;

-- 快照读（直接SELECT的读）
select * from table where ? 
```
在RR中，[innoDB幻读](https://juejin.cn/post/7134186501306318856)通过MVCC避免了快照读的幻读，但是不能避免当前读的幻读

## 运行机制
![SELECT执行流程](https://ucc.alicdn.com/pic/developer-ecology/f2af0e115eef445797fa4f71628f619a.png)

1. 通过[[#C/S 通信协议]]建立MySQL连接
2. [[#查询缓存]]，有则直接返回，否则执行下一步
3. *语法解析*，生成解析树->预处理器，再生成新解析树
4. *优化器*，生成执行计划
5. *执行引擎*，查询数据
![sql执行流程](https://images2015.cnblogs.com/blog/701942/201512/701942-20151210224128402-1287669438.png)
数据库连接池：为了提高性能，避免频繁创建和销毁连接。取代了MySQL驱动 ^[在客户端和MySQL服务器之间建立连接，通过TCP/IP协议实现]

### C/S 通信协议
C/S：客户端/服务器
通信机制：
1. 半双工：同一时刻只有一方发送，一方接收。所以一方发消息，另一方必须等对方发完才能响应，不能切成小块发送，也不能流量控制
2. 全双工：同一时刻两方都可发送，两方都接收
3. 单工
### 查询缓存
 ^[在*MySQL8.0*中被删除]
解析sql前，检查这个语句是否在缓存中，命中则直接返回
缓存的数据结构为一个HashMap，通过通过*客户端信息和SQL语句*创建Hash

##### 机制
如果表结构发生改变，如insert, update, truncate, drop等，缓存失效
sql语句必须完全一致，注释、空格等也需要一致

通过内存池实现内存管理（释放与分配），而不通过操作系统管理

##### 缺点
计算Hash需要时间
缓存失效，如果频繁修改表，缓存会失效
sql需要完全一致，不许有空格or注释

### 语法解析和预处理
语法解析：词法、语法分析，判断关键字及顺序正确性，将SQL语句转换成MySQL能够理解的内部格式->解析树
对于`SELECT stuName, age FROM students WHERE age>18 and 1=1;`
1. 词法分析（Lexical Analysis）：
    - 词法分析器（Lexer）将SQL语句拆分成一系列的标记（Tokens），例如关键字（SELECT, FROM, WHERE等）、标识符（students, stuName, age等）、操作符（=）和字面量（1）。
2. 语法分析（Syntax Analysis）：
    - 语法分析器（Parser）根据SQL语法规则，将这些标记组织成一棵抽象语法树（Abstract Syntax Tree, AST）。这棵树表示了SQL语句的结构，例如它表明了`SELECT`子句、`FROM`子句和`WHERE`子句之间的关系。
3. 无用条件去除
	- 对于 `1=1` 恒为真  则直接删掉这个条件

**预处理器**：检查权限、表是否合法等

### 优化器
*选择最优查询路径*：根据成本（I/O成本和CPU成本）选择最佳的索引和查询计划。
*IO成本*:
1. *固定读取成本*:
    - 每次从磁盘读取数据页到内存的成本设定为1。
    - 无论数据页中包含的数据量大小，读取成本保持不变。
2. *数据页加载*:
    - MySQL在访问数据时，会将包含所需数据的数据页加载到内存中。
    - 数据页的加载是基于程序局部性原理，以提高数据访问效率。

*CPU成本*
1. *条件检查*:
    - 数据加载到内存后，MySQL执行条件检查以筛选满足查询条件的记录。
    - 条件检查包括对数据的比较、逻辑运算等CPU密集型操作。
2. *排序操作*:
    - 对于需要排序的查询，MySQL会执行排序算法处理内存中的数据。
    - 排序操作的CPU成本取决于数据量和排序复杂度。
3. *成本累积*:
    - CPU成本与处理的数据行数成正比。
    - 默认情况下，每处理一条记录的CPU成本为0.2。
    - 在处理大量数据时，CPU成本会随着记录数的增加而线性增长。
4. *查询优化*:
    - MySQL的查询优化器在生成执行计划时会考虑CPU成本。
    - 优化器会尝试选择成本最低的查询路径，以提高查询效率。

## 存储结构
![](https://dev.mysql.com/doc/refman/8.0/en/images/innodb-architecture-8-0.png)
内存结构
磁盘结构
1. 系统表空间system tablespace
2. 独立表空间file-per-table tablespace
3. 通用表空间 general tablespace
4. 临时表空间 temporay tablespace
5. 双写缓冲文件 doublewrite buffer files

### 页
页结构，存放表中数据记录的页，称为索引页或数据页
页大小通常16KB


MySQL 中B+树每个节点都是一个页，叶节点存储数据，非叶子节点存储索引值和页偏移量
B+树*指针大小为6字节*，若主键的类型为bigint，长度为8字节
一个索引页中的索引数为：
$${16 \text{ (KB)} \over 6+8\text{ (B)}} = {16*1024 \over 14} = 1170 $$
即1170个索引

### 缓冲池
> [!info] 与查询缓存的区别
> 查询缓存：在服务层，存储查询的结果
> 缓冲池：在存储引擎层，把磁盘的部分热点数据页放在内存中，

缓存表数据与索引数据；使用LRU淘汰策略；存储在内存中，实现加速读/写，减少磁盘IO
#### 实现原理
##### 预读失效
预读就是提前把页放入缓冲池，以提高性能。预读了但没有真正读取数据，还产生了LRU替换，称为*预读失效*
*线性预读*：连续读取的页数超过一定阈值(X:=64)，就预读后面的页
SQL解决方法：预读时不直接放在LRU中，放在LRU的中间部分，如果访问了就放到头部，否则一点一点往后移

##### 缓冲池污染
全表扫描或批量扫描时，打满了LRU，可能造成性能下降（既要IO还要LRU替换）

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/25/16b8cf6bf4aa4dbc~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)
老数据停留时间窗口机制，老数据被访问不会立马放到新数据中，而是需要满足在老数据区*停留的时间*超过阈值，才放入新数据区。
`innodb_old_blocks_time`老生代停留时间窗口，单位是毫秒，默认是 1000，即同时满足 “被访问” 与“在老生代停留时间超过 1 秒”两个条件，才会被插入到新生代头部。

#### Change Buffer
Change Buffer：写缓冲
https://juejin.cn/post/6844903875271475213
https://zhuanlan.zhihu.com/p/484772808
因为缓冲池，在修改数据时，可是使用*日志先行*
在真正写数据前，记录Redo Log日志，定期使用Check Point技术将Redo Log写入磁盘。且*不会有一致性问题*：
1. 读取会命中缓冲池
2. 缓冲池LRU淘汰会刷脏页回磁盘
3. 数据库崩溃可以通过redo log恢复（还是一致的

其对于*非唯一索引*不在缓冲池的页，执行DML操作不会读磁盘，而是只记录变更(changes)，在未来读取数据时，合并(merge)变更到缓冲池。*减少了磁盘IO*
如果在缓冲池中，就直接更新缓冲池

对于*唯一索引*的DML操作，因为要验证唯一性，所以要扫描整个表（这个字段整个表里面都不能重复！）也是加载到缓冲池，然后更新缓冲池

![[Pasted image 20240517235408.png]]

更新完缓冲池后不直接存磁盘，而是将操作记录到[[#Redo Log]]，空闲时写磁盘

因此*适宜场景*有：
1. 大部分索引是非唯一索引
2. 更新/插入数据后不会立即读取（写多读少，ps:要不为啥叫写缓冲

### 磁盘结构
#### 表空间
1. 系统表空间：存储Change Buffer的数据，是共享表空间
2. 独立表空间：存储单个表的数据和索引，存储在单个文件中
3. 通用表空间：为多个表存储数据，是共享表空间，占用更小的磁盘空间
4. Undo表空间：存储undo log 
5. 临时表空间：执行复杂查询（GROUP BY、ORDER BY、UNION等）时，执行计划中包含Using Temporay字段时创建临时表
#### 双写缓冲 Doublewrite Buffer
刷脏页[^1]时，写磁盘不是原子性的，存在部分写失效[^2]的问题，如果此时宕机没写完数据，磁盘上的数据写了一半，此时数据全乱了，RedoLog也无法解决（RedoLog只能从SQL语句层面恢复，而不能从数据层面恢复），所以我们宁愿没写入磁盘也不愿意写了一半

通过*双写缓冲*解决这个问题：
写数据文件（独立表空间）前，先把他写入双写缓冲文件（在磁盘上）中，然后才写入数据文件，如果在写数据文件时出意外，可以通过双写缓冲中的副本恢复

过程：Change Buffer的数据不直接写入数据文件，而是copy到内存中的Doublewrite Buffer中，然后再拷贝至双写缓冲文件中，每次写入1M，等copy完成后，再将Doublewrite Buffer中的页写入磁盘文件。


[^1]: 刷脏页：将数据库中修改过但尚未写入磁盘的数据页刷入磁盘的过程
[^2]: 写失效：数据在缓冲池时机器宕机。恢复时，可以先读双写缓冲区的数据，然后通过RedoLog来进行恢复。部分写失效：由于InnoDB的页大小(16KB)和OS的页大小(4KB)不一致，在缓冲池在写磁盘的时候宕机，可能只写了InnoDB页的一部分

#### Redo Log
用于数据恢复，宕机重启后重做操作。
先写Log Buffer，然后定时刷入磁盘的Redo Log File。

写前日志(Write Ahead Logging, WAL)策略：先写日志，然后写磁盘。
使用WAL可以将磁盘的随机IO（写独立表空间）变成顺序IO（顺序追加Redo Log File）

`innodb_flush_log_at_trx_commit` 控制redo log的刷盘策略
0：每秒写log buffer到log file，并同时刷入磁盘
1：每次提交写log buffer到log file，并同时刷入磁盘
2：每次提交写log buffer到log file，每秒刷入磁盘 ^c92559

##### redo log 文件组
类似环形数组，两个指针记录当前已经刷入磁盘的位置和最新redo log的位置


#### Undo Log
在更新数据前，会提前生成一个undo log，写入Buffer，后刷入磁盘，在回滚时读取。
就是MVCC中的![[#undo log]]

### 双1设置
数据安全相关的两个关键参数`innodb_flush_log_at_trx_commit` 和 `sync_binlog`，都设为1就是*双1设置*

![[#^c92559]]


`sync_binlog` 控制binlog策略
0：不主动刷入磁盘，交给操作系统
N：每写N次binlog，就刷入磁盘


# 语法
### DISTINCT
- [1148. 文章浏览 I](https://leetcode.cn/problems/article-views-i/)
- [180. 连续出现的数字](https://leetcode.cn/problems/consecutive-numbers/)
### ORDER BY
- [1280. 学生们参加各科测试的次数](https://leetcode.cn/problems/students-and-examinations/)

### CASE
```sql
CASE WHEN condition THEN result
[WHEN...THEN...]
ELSE result
END
```

- [1934. 确认率](https://leetcode.cn/problems/confirmation-rate/)
- 
## 函数
### length 函数
`char_length(str)`：计算字符数，不论数字、字母、汉字
`length(str)`：计算字节数
数字 和 字母算1字节
utf-8：汉字3字节
gbk：汉字2字节

### DATE
[日期函数](https://blog.csdn.net/xyh930929/article/details/119081675)
- [197. 上升的温度](https://leetcode.cn/problems/rising-temperature/)

```sql
DATE_ADD('2021-01-02', INTERVAL 1 QUARTER) ,
DATE_ADD('2021-01-02', INTERVAL 1 MONTH) ,
DATE_ADD('2021-01-02', INTERVAL 1 WEEK) ,
DATE_ADD('2021-01-02', INTERVAL 1 DAY) ,
DATE_ADD('2021-01-02', INTERVAL 1 HOUR) ,
DATE_ADD('2021-01-02', INTERVAL 1 MINUTE) ,
DATE_ADD('2021-01-02', INTERVAL 1 SECOND) ,
```

### ROUND() 舍入
```sql
SELECT ROUND(column_name,decimals) FROM TABLE_NAME;
```
decimals 默认为0

### AVG() / SUM()
计算输入的一组的值的avg/sum

- [1075. 项目员工 I](https://leetcode.cn/problems/project-employees-i/)
- 

## 过滤、连接、分组
[SQL中on与where与having的区别与执行顺序](https://blog.csdn.net/HD243608836/article/details/88813351)

执行顺序： ON -> WHERE -> HAVING
WHERE 相当于 inner join，所以会比on慢一点

HAVING是在聚集函数计算结果出来之后筛选结果，WHERE是在计算之前筛选结果

### JOIN ON

1. 内连接：查询A B交集
	- **隐式**  select emp.name from emp, dept `where emp.dept_id = dept.i`
	- **显式**  select emp.name from emp `inner join dept on` emp.dept_id = dept.id
2. 左连接 ： 查询A表所有，以及A B交集
	- select emp.name from emp `left join dept on` emp.dept_id = dept.id
3. 右连接 ： 查询B表所有，以及A B交集
	- select dept.name from dept `right join emp on` emp.dept_id = dept.id
4. 自连接：查询表与自身的连接，必须使用表别名
	- select a.name b.name `from emp a, emp b where a.id = b.manager_id`

- [1378. 使用唯一标识码替换员工ID](https://leetcode.cn/problems/replace-employee-id-with-the-unique-identifier/)
- [1280. 学生们参加各科测试的次数](https://leetcode.cn/problems/students-and-examinations/)

### HAVING
- [570. 至少有5名直接下属的经理](https://leetcode.cn/problems/managers-with-at-least-5-direct-reports/)


### GROUP BY
- [1581. 进店却未进行过交易的顾客](https://leetcode.cn/problems/customer-who-visited-but-did-not-make-any-transactions/)
- [1280. 学生们参加各科测试的次数](https://leetcode.cn/problems/students-and-examinations/)

### UNION
拼接两个查询结果，只会拼接两个查询不同的值
使用`union all`拼接相同的查询

- [1789. 员工的直属部门](https://leetcode.cn/problems/primary-department-for-each-employee/)
- [1341. 电影评分](https://leetcode.cn/problems/movie-rating/)






### OVER(PARTITION BY ...)
[统计同时在线人数](https://www.bilibili.com/video/BV1iQ4y1L7my)

