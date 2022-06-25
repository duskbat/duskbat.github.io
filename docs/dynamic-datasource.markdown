
# 动态数据源事务不生效问题
## 0x0 背景
因为业务方面的原因，系统中需要跨数据源，技术方案选择了 baomidou/dynamic-datasource-spring-boot-starter.
按照惯例，依赖如下：

| 依赖                                   | 版本  |
| -------------------------------------- | ----- |
| mybatis                                | 3.5.9 |
| mybatis-spring                         | 2.0.7 |
| HikariCP                               | 4.0.3 |
| dynamic-datasource-spring-boot-starter | 3.5.1 |


## 0x1 问题描述
原生事务注解 @Transactional 下事务不生效。
代码大致如下：
```java
@Override
@Transactional(rollbackFor = Exception.class)
public Integer delete(Long id) {
    int i = aaaDao.deleteById(id);
    throw new RuntimeException();
    return i;
}
```

开启 Spring debug log 如下：
```log

2022-06-25 13:08:04.684 DEBUG 7624 --- [io-18600-exec-3] o.s.jdbc.support.JdbcTransactionManager  : Creating new transaction with name [com.ClassName.function]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.lang.Exception
2022-06-25 13:08:04.706 DEBUG 7624 --- [io-18600-exec-3] o.s.jdbc.support.JdbcTransactionManager  : Acquired Connection [HikariProxyConnection@1542466177 wrapping com.mysql.cj.jdbc.ConnectionImpl@46c5747f] for JDBC transaction
2022-06-25 13:08:04.706 DEBUG 7624 --- [io-18600-exec-3] o.s.jdbc.support.JdbcTransactionManager  : Switching JDBC Connection [HikariProxyConnection@1542466177 wrapping com.mysql.cj.jdbc.ConnectionImpl@46c5747f] to manual commit
Creating a new SqlSession
Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@41c2090]
2022-06-25 13:08:04.774 DEBUG 7624 --- [io-18600-exec-3] o.s.jdbc.datasource.DataSourceUtils      : Fetching JDBC Connection from DataSource
JDBC Connection [HikariProxyConnection@162473161 wrapping com.mysql.cj.jdbc.ConnectionImpl@3f4f5321] will be managed by Spring
Fetched SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@41c2090] from current transaction
==>  Preparing: delete from table_aaa where id=?
==> Parameters: 123
<==    Updates: 1
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@41c2090]
Transaction synchronization deregistering SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@41c2090]
Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@41c2090]
2022-06-25 13:08:13.178 DEBUG 7624 --- [io-18600-exec-3] o.s.jdbc.support.JdbcTransactionManager  : Initiating transaction rollback
2022-06-25 13:08:13.178 DEBUG 7624 --- [io-18600-exec-3] o.s.jdbc.support.JdbcTransactionManager  : Rolling back JDBC transaction on Connection [HikariProxyConnection@1542466177 wrapping com.mysql.cj.jdbc.ConnectionImpl@46c5747f]
2022-06-25 13:08:13.191 DEBUG 7624 --- [io-18600-exec-3] o.s.jdbc.support.JdbcTransactionManager  : Releasing JDBC Connection [HikariProxyConnection@1542466177 wrapping com.mysql.cj.jdbc.ConnectionImpl@46c5747f] after transaction
```

## 0x2 问题溯源
可以看见，日志里面很干净，整个事务执行的过程都有：
- 创建事务
- 获取数据库连接
- 切换为手动提交
- 创建 SqlSession
- 注册事务同步
- Fetching JDBC 连接 from DataSource
- JDBC 连接 托管给 Spring
- Fetched SqlSession
- 执行SQL
- 释放 SqlSession
- 事务注销 SqlSession
- 事务关闭 SqlSession
- 初始化事务回滚
- 回滚
- 释放连接
- 抛出异常

查看一下 mysql 的日志：
```sql
-- 开启 mysql 日志
show variables like '%log_output%';
show variables like '%general_log%';
SET GLOBAL log_output = 'TABLE';
SET GLOBAL general_log = 'ON';
-- 查看 日志
select *
from mysql.general_log
where user_host = 'dev[dev] @  [127.0.0.1]'
order by event_time desc;
-- 数据太多可以清一下
truncate table mysql.general_log;
```

| argument                               |
| -------------------------------------- |
| SET autocommit=1                       |
| rollback                               |
| DELETE FROM	table_aaa WHERE	id = '123' |
| SELECT @@session.transaction_read_only |
| SET autocommit=0                       |

并没有发现什么问题，还是该有的都有，rollback 已经执行了。

## 0x3 胡乱分析
一个程序员面临未知的 bug 却没有头绪怎么办？当然是打断点调试。
在一阵胡乱操作之后，发现了一些端倪。
首先回到日志，在日志里，有两个 JDBC Connection:
- HikariProxyConnection@1542466177
- HikariProxyConnection@162473161

显然，这俩并不是同一个 JDBC Connection。

## 0x4 初步分析
对 spring-tx 有所了解的都知道，在事务中，Connection 会被 ConnectionHolder 持有，然后通过 TransactionSynchronizationManager.bindResource() 扔进 ThreadLocal 里面缓存。
使用的时候通过 TransactionSynchronizationManager.getResource() 取出。
```java
public abstract class TransactionSynchronizationManager {
	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");

	private static Object doGetResource(Object actualKey) {}        
}
```
查看调用的堆栈可以发现，形参 actualKey 是 DataSource 类型.
进一步调试可以发现，获取 connection 的两个入口：
- DataSourceTransactionManager.getDataSource()
- DataSourceUtils.getConnection(DataSource dataSource)

他俩的 dataSource 分别是：
- DataSourceTransactionManager.dataSource = ItemDataSource
- SpringManagedTransaction.dataSourse = DynamicRoutingDataSource

| call                                                               | referrance                              | dataSource               |
| ------------------------------------------------------------------ | --------------------------------------- | ------------------------ |
| DataSourceTransactionManager.getDataSource()                       | DataSourceTransactionManager.dataSource | ItemDataSource           |
| DataSourceUtils.getConnection(SpringManagedTransaction.dataSource) | SpringManagedTransaction.dataSourse     | DynamicRoutingDataSource |


## 0x5 深入分析
为什么数据源会不一样？
咱先查一查 DataSourceTransactionManager 和 SpringManagedTransaction 的数据源是怎么初始化的。
- DataSourceTransactionManager 是从 DataSourceTransactionManagerAutoConfiguration 初始化的，dataSource 从 org.apache.ibatis.mapping.Environment 取的
- SpringManagedTransaction 是从 SpringManagedTransactionFactory 初始化的， dataSource 是从 Spring 容器里取的

两个数据源都是 DynamicRoutingDataSource !
那是谁把 DataSourceTransactionManager 的数据源 改成 ItemDataSource 的?

通过调试发现，项目里面有这么一段代码：
```java
@Bean
public SpringProcessEngineConfiguration springProcessEngineConfiguration(DataSourceTransactionManager dataSourceTransactionManager) {
    DataSource flowable = dataSource.getDataSource("master");
    SpringProcessEngineConfiguration springProcessEngineConfiguration = new SpringProcessEngineConfiguration();
    springProcessEngineConfiguration.setDataSource(flowable);
    dataSourceTransactionManager.setDataSource(flowable);
    springProcessEngineConfiguration.setTransactionManager(dataSourceTransactionManager);
    springProcessEngineConfiguration.setDatabaseSchemaUpdate(ProcessEngineConfiguration.DB_SCHEMA_UPDATE_FALSE);
    return springProcessEngineConfiguration;
}
```
到这里问题就很明确了。。。