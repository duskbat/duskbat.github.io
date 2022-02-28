---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: page
title: 浅析MySQL之：锁


---
# 锁
对数据库的并发访问和数据一致性之间存在天然的矛盾, 因此有了锁机制。这也是数据库系统区别于文件系统的重要原因之一。

## 什么是锁
如果给锁一个定义的话: 锁机制是为了支持对共享资源进行并发访问。  

值得一提的是, 数据库中的锁不仅包括面向事务的lock, 也包括面向其他共享资源的latch. 如操作缓冲池中的LRU列表的元素时就需要锁(latch)保证一致性。

另外, 数据库之间、存储引擎之间对于锁的实现完全不同。本文主要介绍InnoDB对于锁的实现。

## latch 和 lock
latch的对象是内存中的共享资源, 以保证并发线程正确操作临界资源。latch可以分为mutex(互斥锁)和rwlock(读写锁)。

lock的对象是事务, 以锁定数据库中的对象(如表、页、行)。lock通常仅在事务提交或回滚后进行释放, 且有死锁检测机制。

通过如下命令可以查看InnoDB中的latch:
```sql
SHOW ENGINE INNODB MUTEX;
```

## InnoDB 中的 lock
按锁粒度可分为: 
- 行锁
  - 共享锁(S Lock): 允许事务读一行数据; 可以与其他S锁兼容。
  - 排他锁(X Lock): 允许事务更新或删除一条数据; 不与其他行锁兼容。
- 表锁
- 意向锁
  - 意向共享锁(IS Lock): 事务想要获得一张表中某几行的共享锁
  - 意向排他锁(IX Lock): 事务想要获得一张表中某几行的排他锁

**意向锁**  
InnoDB支持多粒度(granular)锁定, 允许事务在行级和表级上的锁同时存在。  
为了支持在不同粒度上进行加锁操作, InnoDB使用了意向锁(Intention Lock): 如果想要对最细粒度的对象(例如行)上锁, 那么需要先对数据库、表、页上意向锁, 然后才能对行上锁。
InnoDB中的意向锁是表级锁。

**InnoDB中锁不兼容情况**  
因为行级锁的存在, 所以意向锁不会阻塞除全表扫描意外的任何请求。
> 意向锁之间是互相兼容的。
> IX和S不兼容, 有X的全都不兼容。

|     |     |     |     |     |
| --- | --- | --- | --- | --- |
|     | IS  | IX  | S   | X   |
| IS  |     |     |     | N   |
| IX  |     |     | N   | N   |
| S   |     | N   |     | N   |
| X   | N   | N   | N   | N   |


可以通过命令查看当前有关锁的信息: 
```sql
SHOW ENGINE INNODB STATUS \G;
```
从InnoDB 1.0之后，通过 INFORMATION SCHEMA 中的三张表可以更好地监控当前事务, 并分析可能存在的锁问题:
- INNODB_TRX
- INNODB_LOCKS
- INNODB_LOCK_WAITS

## 一致性读
**一致性非锁定读**  
在read committed和repeatable read隔离级别下，一致性非锁定读是InnoDB默认的读取方式，其实现方式是MVCC。  
这种读取方式通过读取undo的快照数据，避免了加锁操作，同样也不需要等待锁的释放。  
在 read committed 和 repeatable read 隔离级别下，都是使用一致性非锁定读，但是对于版本的定义是不同的。
- read committed: 读取被锁定行的最新版本 (其实是违反了 I 隔离性)
- repeatable read：读取事务开始时的版本

**一致性锁定读**  
除了非锁定读外，可以显式地对数据库读取操作进行加锁。  
InnoDB对SELECT语句支持两种一致性锁定读操作：
- SELECT ... FOR UPDATE
    > 加Ｘ锁
- SELECT ... LOCK IN SHARE MODE
    > 加Ｓ锁

使用以上两种锁定语句的时候必须保证在一个事务中进行，事务的提交会释放锁。


## 锁的算法
InnoDB有三种行锁的算法
- Record Lock: 单个行记录上锁，锁住索引记录
- Gap Lock: 间隙锁
- Next-Key Lock: Record Lock + Gap Lock

Next-Key Locking是为了解决幻影行的问题。这里注意，在repeatable read隔离级别下非锁定读也解决了幻行的问题。因为MVCC中SELECT会查找行版本号早于当前事务版本号的数据行，INSERT会赋予新插入的行以当前事务版本号作为行版本号。
> 举例来讲，事务A开始，版本号A；然后事务B开始，版本号B，事务B插入数据行，版本号为B，这一行对A是不可见的。

除了next-key还有previous-key，Next-Key是前开后闭区间，previous-key是前闭后开区间。当执行INSERT操作的时候会检测下一个索引有没有锁定，如果锁定则阻塞。  
当有唯一索引的时候，next-key Lock会降级为Record Lock。

幻行指的是同一事物中，连续两次查询会返回不同的结果。  
关于幻行，根据上述的意思应该可以得出，行指的是索引列相同的行记录。

Gap Lock可以通过两种方式显式关闭：
- read committed 隔离级别
- 参数 innodb_locks_unsafe_for_binlog 设为 1

这样除了唯一性检查之外(还有外键)就没有其他的Gap Lock了。  
关闭Gap Lock会破坏事务的隔离性，并且可能会导致主从数据不一致。此外，性能上也没有优势。  
可以利用 Next-key Lock 可以进行唯一性检查。  
> 有读者可能会有疑问，SELECT ... LOCK IN SHARE MODE 会加S锁，这时候并发操作，insert的时候会不会有问题？
> 其实不会的，只有一个insert能成功，其他的都会抛出死锁的错误。

## 锁问题
古典的三种问题分别是 脏读、不可重复读、丢失更新

### 脏读
要理解脏读首先要明确脏数据的概念，这里的脏数据跟脏页是不同的两种概念。脏页指的是缓冲池中已经被修改的页，还没有刷写到磁盘(当然redo日志已经记录了)；而脏数据是事物对缓冲池中行记录的修改，并且还未提交。  
read uncomitted 会直接导致脏读，这违反了隔离性。
> read uncomitted 在一些比较特殊的场景还是有用的，例如 replication 环境中的slave节点，并且slave上的查询不需要精确的返回值。

### 不可重复读
不可重复读指的是一个事物内多次读同一数据集合，在该事务的过程中其他事务的DML操作会导致该事务的读取产生不一致。  
read commited 会导致不可重复读，这也是违反了隔离性。  
在MySQL官方文档中不可重复读被定义为了Phantom Problem。在InnoDB中，通过临键锁解决了不可重复读问题。  
> 《高性能MySQL》和《MySQL技术内幕-InnoDB存储引擎》对该部分的解释有所差异，我倾向于《MySQL技术内幕》中的解释，请查阅MySQL官方文档自己明辨。

### 丢失更新
也是经典的ABA问题，不过在数据库中有锁机制，更新操作是不会出现ABA问题的。  
但是在业务逻辑中却可能产生这样的情况，先查询再修改的场景如果查询操作没加X锁，会导致数据不一致，也就是业务意义上的“帐不平”。

## 阻塞
首先给出阻塞的定义：因为锁之间的兼容性，锁的等待其他锁释放该锁需要的资源就是阻塞。  
InnoDB中，参数 innodb_lock_wait_timeout 控制等待时间(默认50s), innodb_rollback_on_timeout 设定等待超时是否进行回滚(默认OFF)。  
等待时间是动态生效的，是否回滚设置是静态的，启动后只读。超时抛出 ERROR 1205。  
> 需要注意的一点是，InnoDB在很多时候都不会对异常进行回滚，而且既不commit也不rollback，比如一个事务中插入两条数据，第二条由于阻塞超时了，第一条是不回滚的。

## 死锁
死锁指的是两个以上事务在执行过程中，因争夺锁资源而造成的相互等待的现象。  
解死锁的思路很简单，就是让其中一个事务先释放锁资源，通常是事务回滚。  
最简单的解决方案是根据等待时间超时回滚，这本质上是根据FIFO顺序进行回滚，如果超时的事务操作很重，更新了很多行，占用了很多的undo log, 这种方式可能就是很不恰当了。  
因此，数据库普遍采用等待图(wait for graph)进行死锁检测。等待图需要保存两种信息：
- 锁的信息链表
- 事务等待链表

链表都是有先后顺序的，事务间的等待会画出一条边，如果等待图有环，那么就存在死锁。

| 事务等待链表 |     | row1锁 链表 |     | row2锁 链表 |
| ------------ | --- | ----------- | --- | ----------- |
| t1           |     | t2:x        |     | t1:s        |
| t2           |     | t1:s        |     | t4:s        |
| t3           |     |             |     | t2:x        |
| t4           |     |             |     | t3:x        |

等待图如下：  
```mermaid
graph LR
t1-->t2
t2-->t1
t2-->t4
t3-->t1
t3-->t2
t3-->t4
```

显然，t1和t2中有环，存在死锁，通常InnoDB会回滚undo量最小的事务。
> 死锁错误ERROR 1213, InnoDB会马上回滚。

## 锁开销
InnoDB对于锁的实现是采用位图实现的。每个页都有位图，一个事务对一条记录加锁和对一整页的事务加锁开销是一样的。
