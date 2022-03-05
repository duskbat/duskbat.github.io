# <center>2. MySQL</center>

### SQL 调优  
思路: 面向响应时间的优化, 首先要明确时间花在什么地方, 做剖析(profiling) 
``` 
set profiling=1;
show profiles;
show profile cpu for query 1;

    +--------------------------------+----------+
    | Status                         | Duration |
    +--------------------------------+----------+
    | starting                       | 0.000053 |
    | Executing hook on transaction  | 0.000003 |
    | starting                       | 0.000006 |
    | checking permissions           | 0.000003 |
    | Opening tables                 | 0.000139 |
    | init                           | 0.000003 |
    | System lock                    | 0.000005 |
    | optimizing                     | 0.000003 |
    | statistics                     | 0.000008 |
    | preparing                      | 0.000011 |
    | executing                      | 0.000032 |
    | end                            | 0.000002 |
    | query end                      | 0.000002 |
    | waiting for handler commit     | 0.000005 |
    | closing tables                 | 0.000004 |
    | freeing items                  | 0.000027 |
    | cleaning up                    | 0.000010 |
    +--------------------------------+----------+
```
**服务器层结构**:
- **连接器**: 身份认证和权限相关(登录 MySQL 的时候)。
- **查询缓存**: 执行查询语句的时候，会先查询缓存（MySQL 8.0 版本后移除，因为此功能不实用）。
- **分析器**: SQL 语句会经过分析器，分析器说白了就是要先看你的 SQL 语句要干嘛，再检查你的 SQL 语句语法是否正确。
- **优化器**: 按照 MySQL 认为最优的方案去执行。
- **执行器**: 执行语句，然后从存储引擎返回数据。

**更新语句执行过程**:
1. 从磁盘文件中找到对应查询条件的整页数据，加载到buffer pool
2. 将更新数据的旧值写入到undo log文件备份
3. 更新buffer pool内的数据，写redo log buffer
4. 准备提交事务, 准备将redo log写入磁盘，redo log prepared
5. 执行器生成binlog, 并准备将binlog写入磁盘
6. 执行器调用存储引擎接口，写入commit标记到redo log里，redo log commit，提交事务完成，buffer pool随机写入磁盘。

> 二阶段提交: 先redo log prepared, 然后记录 binlog, 最后redo log commit

### 索引没有被使用
- 隐式类型转换
- 如果where条件中含有or, 除非or条件中的所有列都是索引列，否则不走
- 对于多列索引，如果没有使用前导列
- 如果对索引字段使用函数算数运算或者其他表达式操作
- 数量少，觉得全表扫描比用索引快
- like以%开头


### 隐式类型转换  
- varchar 类型的会隐式转换为数值型
- 例如 varchar 类型的索引, 查询条件用 int 索引失效  
- 隐式转换 varchar -> int 截取前n个数字形式的字符，如果n=0，则转换为0;  

### InnoDB 如何实现事务的 ACID
- 使用 undo log(回滚日志) 来保证事务的原子性。
- 使用 redo log(重做日志) 保证事务的持久性。
- InnoDB 通过 锁机制、MVCC 等手段来保证事务的隔离性（ 默认支持的隔离级别是 REPEATABLE-READ ）。
- 保证了事务的持久性、原子性、隔离性之后，一致性才能得到保障。

### InnoDB 行级锁的实现
- Record lock：单个行记录上的锁
- Gap lock：间隙锁，锁定一个范围，不包括记录本身
- Next-key lock：record+gap 锁定一个范围，包含记录本身

### MySQL 缓存问题
- MySQL 8.0后移除; 
- 可以手动开启; 
- 可以设置大小(几十MB比较合适); 
- 可以控制某个具体查询语句是否需要缓存;

### 事务的ACID
- 原子性(Atomicit):     事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
- 一致性(Consistency):  执行事务前后，数据保持一致，例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；
- 隔离性(Isolation):    并发访问数据库时，事务不被其他事务所干扰，各并发事务之间是独立的；
- 持久性(Durabilily):   一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。


### 事务隔离级别  
- READ UNCOMMITIED（读未提交）
  - 一个事务未提交时，对其他事务也是可见的
  - 脏读
- READ COMMITIED（读已提交） 
  - 第一个事务中的两次读数据之间，由于第二个事务的提交导致第一个事务两次读取的数据可能不一样
  - 不可重复读
  - 大多数DB系统的默认隔离级别。
- REPEATABLE READ（可重复读）
  - 读事务开始时的数据
  - 幻读
  - 其他事务增删操作会产生幻行，InnoDB和XtraDB通过多版本并发控制(MVCC)解决了幻读。
- SERIALIZABLE（串行化）
  - 事务串行，读取的每一行数据上都加锁，可能会造成大量的超时和锁争用问题，只有非常需要确保数据的一致性而且可以接受没有井发的情况下才考虑

### InnoDB怎么解决幻读问题
- 在当前读场景下通过LBCC(基于锁的并发控制)
  基于锁的并发控制, 读取的时候加临键锁(next-key). 
  > select for update || DELETE\UPDATE\INSERT INTO\REPLACE INTO || SELECT LOCK IN SHARE MODE
- 在快照读场景下通过MVCC

### MVCC
InnoDB MVCC 的实现是通过保存数据在某个时间点的快照来实现的。
不管事务需要执行多长时间，它看到的数据是一致的；根据事务的开始时间不同，不同事务在同一时刻看到的数据可能是不一样的。  
只在读已提交和可重复读下工作  
**InnoDB**:
- 行记录版本号：每个行记录保存两个版本号(创建、删除)
- 事务版本号：每开始一个新的事务，版本号自增。事务开始时刻自增的版本号作为事务的版本号。
- 在可重复读的隔离级别下：
  - select (核心：事务开启的时刻，记录是最新存在的)
    - 只查找版本**早于或等于**当前事务版本的数据行，确保事务读取的行早于事务或事务本身插入或修改的。
    - 行的过期版本要么未定义，要么**大于**当前事务版本号，确保事务读取到的行，在事务开始之时未过期。
    - 只有条件1、2同时满足的记录，才能返回作为查询结果.

  - insert
    - 新增时保存当前系统版本号作为新增版本号。

  - delete
    - 删除时保存系统当前版本号作为删除版本号。
  
  - update
    - 插入一条新纪录，当前系统版本号作为新增版本号，同时保存当前系统版本号作为原记录的删除版本号。

- 个人理解: 在并发条件下, 比如事务A(版本1) 事务B(版本2):
  - (B的增删操作影响不到A) B删除，A也能读到, 但不能读B的新增, B更新, A读到原纪录.
  - (A的增删操作会影响B) B能读A的新增, A和自己的删除就读不到了, A更新, B读到新记录.

### MySQL索引两种主要的数据结构
- 哈希索引
  最大的缺点是不支持**顺序和范围**查询
- BTree
  都是使用的B+Tree
  - MyISAM
    叶节点的data域存放的是数据文件的地址; 非聚簇索引
  - InnoDB
    主键索引本身包含了全部数据，其他的辅助索引的叶节点存储的是主键; 聚簇索引
- 空间数据索引
- 全文索引

### 覆盖索引
包含查询中所有字段的索引
不需要回表, 减少一次索引操作, 随机I/O比顺序I/O慢

### 注意区分数据的物理结构和逻辑结构 todo
物理结构：（段segment 区extent 页 行）
逻辑结构： 
### 索引为什么快
MySQL基本存储结构 (InnoDB引擎)
将数据划分为若干个磁盘页，以页作为磁盘和内存之间交互的基本单位，InnoDB中页的大小一般为 16 KB。通常情况下，一次最少从磁盘中读取16KB的内容到内存中，一次最少把内存中的16KB内容刷新到磁盘中。
- 各个数据页之间可以组成一个双向链表（就是B+树的各个页之间都按照索引值顺序用双向链表连接起来）
- 每个数据页都会为存储在它里边的记录生成一个页目录，该目录是用数组进行管理，在通过主键查找某条记录的时候可以在页目录中使用二分法快速定位到对应的槽，然后再遍历该槽对应分组中的记录即可快速找到指定的记录

无索引条件下O(n)复杂度，索引重新组织了逻辑顺序, 使查找数据页的时候复杂度变成O(logN)

### B树和B+树的区别
**B**:
- 所有节点不仅存放key,也存放data
- 所有的叶节点都是独立的
- 检索过程都是类似二分查找, 有时候不到叶子节点就找到了
 
**B+**:
- 只有叶子节点存放key和data, 其他只存放key
- 叶子节点有一条引用链指向相邻的叶节点
- 任何查找都是从根到页, 页之间顺序检索很明显


### binlog
binlog 记录了执行更改(只要执行就会记录，并不是非要实际更改)的所有操作
- 恢复
- 复制
- 审计：核验是否有注入攻击

事务未提交的日志会记录到缓存，提交时将缓存写入日志文件。
sync_binlog 每多少次事务将缓存写入，1是即时写，但是会有性能影响，并且事务未提交会出现脏数据。
binlog_format: STATEMENT / ROW / MIXED
STATEMENT 不支持并发 时间函数问题
ROW 支持并发

### 跳表与B+树
MySQL的性能瓶颈通常在IO上，B+树叶子节点磁盘页的设计是为了最大限度地降低磁盘IO
Redis内存读写，使用了跳表

### B树和B+树
树高，减少IO次数；
叶子节点有双向的指针，方便范围查询

### Datetime 和 Timestamp
Datetime 
- 8字节
- 无时区

Timestamp 有些问题，通常用int
- 4字节
- 有时区
- 存储到s, 毫秒可用MariaDB, 或额外用字段存

### 读写分离
主要是为了将对数据库的读写操作分散到不同的数据库节点上, 小幅提升写性能，大幅提升读性能。
实现:
- 部署多台数据库，选择一种的一台作为主数据库，其他的一台或者多台作为从数据库。
- 保证主数据库和从数据库之间的数据是实时同步的，这个过程也就是我们常说的主从复制。
- 系统将写请求交给主数据库处理，读请求交给从数据库处理。

**主从同步过程**
```
主库写binlog -> 从库请求更新binlog -> 主库发送 -> 从库接收 -> 写到relay log -> 执行
```
- 主库将数据库中数据的变化写入到 binlog
- 从库连接主库
- 从库会创建一个 I/O 线程向主库请求更新的 binlog
- 主库会创建一个 binlog dump 线程来发送 binlog ，从库中的 I/O 线程负责接收
- 从库的 I/O 线程将接收的 binlog 写入到 relay log 中。
- 从库的 SQL 线程读取 relay log 同步数据本地（也就是再执行一遍 SQL ）。

**主从同步延迟解决**:
- 把从路由到主库
- 延迟读取时间

### 分库分表
- 单表数据大 
- 数据库太大 
- 应用并发太大

**解决方案**:
Apache ShardingSphere 

### <center>explain</center>  
```
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | seckill | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+

  id //越大优先级越高
  select_type  
  table  
  partitions  //分区  
  type  
  possible_keys key key_len  //精度越高len越大
  ref  
  rows  
  filtered  // 是一个百分比的值，rows * filtered/100 可以估算出将要和前一个表进行连接的行数
  Extra  
```

**select_type**:  
- SIMPLE 简单的select查询，查询中不包含子查询或者UNION
- PRIMARY 查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY
- SUBQUERY 在SELECT或WHERE列表中包含了子查询
- DERIVED 在FROM列表中包含的子查询被标记为DERIVED（衍生），MySQL会递归执行这些子查询，把结果放在临时表中
- UNION 若第二个SELECT出现在UNION之后，则被标记为UNION; 若UNION包含在FROM子句的子查询中，外层SELECT将被标记为：DERIVED
- UNION RESULT 从UNION表获取结果的SELECT

**type**
  > system > const > eq_ref > ref > range > index > all  
一般来说，得保证查询至少达到range级别，最好能达到ref。

- system 表只有一行记录（等于系统表），这是const类型的特列，平时不会出现
- const 表示通过索引一次就找到了
- eq_ref 唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配
- ref 非唯一性索引扫描，返回匹配某个单独值的所有行
- range 只检索给定范围的行，使用一个索引来选择行
- index 索引树扫描, 只遍历索引树
- ALL 拉跨

**ref**  
- 哪些列或常量被用于查找索引列上的值,比如id=1,那么ref:const; 显示索引的哪一列被使用了.

**rows**
- 大致估算出, 找到所需的记录需要读取的行数

**Extra** 
- Backward index scan: 优化器能在InnoDB上用降序索引
- Using filesort: 使用了外部索引文件排序, 没有按照表内的索引顺序
- Using temporary: 使用了用临时表保存中间结果，MySQL在对查询结果排序时使用临时表。
  > 常见于 order by和 group by

- Using where: 表示在Server进行了过滤, where 条件中有非索引字段
- Using index: 表示使用了覆盖索引 todo 下面2条说得不对
  - 有 using where: 索引用于查找，需要回表查数据
  - 无 using where: 索引用于读取数据而非执行查找动作
- Using index condition: 索引下推
- Using join buffer: 表明使用了连接缓存. 比如说在查询的时候，多表join的次数非常多，那么将配置文件中的缓冲区的join buffer调大一些。


### 分库分表分区
- 分区，通常指的是分区表的概念
就是把一张表的数据分成N个区块，在逻辑上看最终只是一张表，但底层是由N个物理区块组成的
- 分表
将一张表行级拆分成多张实体表
- 分库(tms_1,2,3,4,5)
单台DB性能不够。
垂直拆分，按业务拆分，跨数据库事务问题
水平拆分，按一些规则做一些路由 (如cityId)


### Union Union All
Union 排序且去重
Union all 不排序不去重

### 交集差集并集
```sql
DROP TABLE IF EXISTS t1 ;
CREATE TABLE t1 (name VARCHAR(30) , age int) ENGINE=innodb;
insert into t1 VALUES ('张三',33);
insert into t1 VALUES ('李四',44);
insert into t1 VALUES ('王五',55);
insert into t1 VALUES ('孙六',66);
 
DROP TABLE IF EXISTS t2 ;
CREATE TABLE t2 (name VARCHAR(30) , age int) ENGINE=innodb;
insert into t2 VALUES ('张三',33);
insert into t2 VALUES ('李四',44);
insert into t2 VALUES ('秦七',77);


-- 交集
select * from t1 
UNION
select * from t2

-- 并集
#方式一
select * from t1 where  EXISTS (
	select * from t2 where t1.name= t2.name and t1.age= t2.age
);
 
#方式二 (不推荐)
select * from t1 where (t1.name ,t1.age) in (
	select * from t2
)
 
#方式三 从全集(包括重复)中找到只出现2次的
select * from (
	select * from t1
union all
	select * from t2
) t1 GROUP BY name,age HAVING COUNT(*)=2

-- 差集
-- 方式1
select * from t1 where not EXISTS (
select * from t2 where t1.name = t2.name and t1.age = t2.age
)

-- 方式2 从全集(包括重复)中找到只出现一次的
select * from (
	select * from t1
union all
	select * from t2
) t1 GROUP BY name,age HAVING COUNT(*)=1

-- 方式3 
select * from t1 
where (name,age) 
not in ( select * from t2)
 
-- 方式4 子集情况
select t1.* from t1 
LEFT JOIN t2 
on t1.name = t2.name 
and t1.age = t2.age 
where t2.name is null 
```

### 锁
行锁、表锁、一致性读（还有线程latch）
行锁：共享锁 排他锁
表锁：意向共享锁 意向排他锁 共享锁 排他锁
一致性读：当前读MVCC和快照读LBCC

------------------------------------
