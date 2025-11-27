[toc]
# MySQL 面试八股文

## 一、MySQL 基础

### 1. MySQL 的架构
**分层结构**：
- **连接层**：客户端连接、线程处理
- **服务层**：查询解析、分析、优化、缓存
- **引擎层**：存储引擎（InnoDB、MyISAM等）
- **存储层**：数据文件、日志文件

### 2. 存储引擎对比

**InnoDB**：
- 支持事务
- 支持外键
- 行级锁
- 支持崩溃恢复
- 默认引擎

**MyISAM**：
- 不支持事务
- 不支持外键
- 表级锁
- 查询速度快
- 适合读多写少

### 3. InnoDB 和 MyISAM 的区别
| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务 | 支持 | 不支持 |
| 外键 | 支持 | 不支持 |
| 锁粒度 | 行锁 | 表锁 |
| 崩溃恢复 | 支持 | 不支持 |
| MVCC | 支持 | 不支持 |
| 全文索引 | 5.6+ 支持 | 支持 |

### 4. 数据类型

**数值类型**：
- TINYINT、SMALLINT、MEDIUMINT、INT、BIGINT
- FLOAT、DOUBLE、DECIMAL

**字符串类型**：
- CHAR、VARCHAR
- TEXT、BLOB
- ENUM、SET

**日期时间类型**：
- DATE、TIME、DATETIME
- TIMESTAMP、YEAR

### 5. CHAR 和 VARCHAR 的区别
- **CHAR**：定长，最大255字符，空间固定，速度快
- **VARCHAR**：变长，最大65535字节，节省空间，需要额外字节存储长度

### 6. DATETIME 和 TIMESTAMP 的区别
- **DATETIME**：8字节，范围1000-9999年，不受时区影响
- **TIMESTAMP**：4字节，范围1970-2038年，受时区影响，自动更新

## 二、索引

### 1. 索引的类型
- **B+Tree 索引**：最常用
- **Hash 索引**：等值查询快
- **全文索引**：FULLTEXT
- **空间索引**：SPATIAL

### 2. 索引的分类
- **主键索引**：PRIMARY KEY
- **唯一索引**：UNIQUE
- **普通索引**：INDEX
- **组合索引**：多列索引
- **前缀索引**：字符串前缀

### 3. B+Tree 的特点
- 所有数据存储在叶子节点
- 叶子节点通过指针连接
- 非叶子节点只存储索引
- 树的高度低，IO次数少
- 范围查询效率高

### 4. 为什么使用 B+Tree 而不是 B Tree
- 所有数据在叶子节点，范围查询更高效
- 非叶子节点不存数据，可以存更多索引，树更矮
- 叶子节点有指针连接，遍历更方便

### 5. 为什么不用红黑树
- 红黑树是二叉树，树高度大，IO次数多
- B+Tree 一个节点可以存储多个元素，减少IO

### 6. 聚簇索引和非聚簇索引
**聚簇索引**：
- 数据和索引存储在一起
- InnoDB 的主键索引
- 一个表只有一个

**非聚簇索引**：
- 索引和数据分开存储
- 叶子节点存储主键值
- 需要回表查询

### 7. 回表查询
- 通过非聚簇索引查询时，先找到主键值
- 再通过主键索引查询完整数据
- 两次索引查询

### 8. 覆盖索引
- 查询的列都在索引中
- 不需要回表
- 提高查询效率

```sql
-- id 是主键，name 有索引
SELECT id, name FROM user WHERE name = 'Tom'; -- 覆盖索引
SELECT id, name, age FROM user WHERE name = 'Tom'; -- 需要回表
```

### 9. 索引下推（ICP）
- MySQL 5.6 引入
- 在索引遍历过程中，先过滤索引中的数据
- 减少回表次数

### 10. 最左前缀原则
- 组合索引从最左列开始匹配
- 不能跳过中间列
- 遇到范围查询停止匹配

```sql
-- 索引：(a, b, c)
WHERE a = 1 AND b = 2 AND c = 3  -- 使用 a,b,c
WHERE a = 1 AND b = 2            -- 使用 a,b
WHERE a = 1 AND c = 3            -- 使用 a
WHERE b = 2 AND c = 3            -- 不使用索引
WHERE a > 1 AND b = 2            -- 使用 a，b不使用
```

### 11. 索引失效的情况
- 使用 != 或 <>
- 使用 IS NULL 或 IS NOT NULL
- 使用 OR（除非所有条件都有索引）
- 使用 LIKE '%xxx'
- 对索引列进行函数操作
- 隐式类型转换
- 使用 NOT IN、NOT EXISTS

### 12. 索引优化建议
- 选择区分度高的列
- 使用短索引（前缀索引）
- 利用覆盖索引
- 避免冗余索引
- 删除未使用的索引
- 考虑索引的维护成本

### 13. 什么时候不需要创建索引
- 数据量小
- 频繁更新的列
- 区分度低的列
- 不在 WHERE、ORDER BY、GROUP BY 中使用的列

## 三、事务

### 1. 事务的四大特性（ACID）
- **原子性（Atomicity）**：全部成功或全部失败
- **一致性（Consistency）**：数据完整性约束
- **隔离性（Isolation）**：事务之间相互隔离
- **持久性（Durability）**：提交后永久保存

### 2. 事务的隔离级别
- **读未提交（READ UNCOMMITTED）**：脏读、不可重复读、幻读
- **读已提交（READ COMMITTED）**：不可重复读、幻读
- **可重复读（REPEATABLE READ）**：幻读（InnoDB 通过 MVCC 和间隙锁解决）
- **串行化（SERIALIZABLE）**：无并发问题，性能最差

### 3. 脏读、不可重复读、幻读
- **脏读**：读到其他事务未提交的数据
- **不可重复读**：同一事务中多次读取同一数据结果不同
- **幻读**：同一事务中多次查询记录数不同

### 4. MySQL 默认隔离级别
- **REPEATABLE READ**（可重复读）
- InnoDB 通过 MVCC 和 Next-Key Lock 解决幻读

### 5. MVCC 原理
**多版本并发控制**：
- 每行记录有隐藏列：事务ID、回滚指针
- 通过 undo log 构建版本链
- 通过 Read View 判断数据可见性
- 实现读不加锁

**Read View**：
- m_ids：活跃事务列表
- min_trx_id：最小活跃事务ID
- max_trx_id：下一个事务ID
- creator_trx_id：当前事务ID

**可见性判断**：
- trx_id < min_trx_id：可见
- trx_id >= max_trx_id：不可见
- min_trx_id <= trx_id < max_trx_id：在 m_ids 中不可见，否则可见

### 6. 当前读和快照读
**快照读**：
- 读取历史版本
- 不加锁
- SELECT

**当前读**：
- 读取最新版本
- 加锁
- SELECT ... FOR UPDATE
- SELECT ... LOCK IN SHARE MODE
- INSERT、UPDATE、DELETE

### 7. undo log 和 redo log
**undo log**：
- 回滚日志
- 保证原子性
- 实现 MVCC

**redo log**：
- 重做日志
- 保证持久性
- 崩溃恢复

### 8. binlog
- 二进制日志
- 记录所有 DDL 和 DML
- 用于主从复制和数据恢复
- Server 层实现

### 9. redo log 和 binlog 的区别
| 特性 | redo log | binlog |
|------|----------|--------|
| 层次 | InnoDB 引擎层 | Server 层 |
| 内容 | 物理日志 | 逻辑日志 |
| 写入方式 | 循环写 | 追加写 |
| 作用 | 崩溃恢复 | 主从复制、数据恢复 |

### 10. 两阶段提交
- 保证 redo log 和 binlog 一致性
- **prepare 阶段**：写 redo log，状态为 prepare
- **commit 阶段**：写 binlog，redo log 状态改为 commit

### 11. 事务的实现原理
- **原子性**：undo log
- **持久性**：redo log
- **隔离性**：锁 + MVCC
- **一致性**：以上三者保证

## 四、锁

### 1. 锁的分类

**按粒度**：
- 全局锁
- 表级锁
- 行级锁

**按性质**：
- 共享锁（S锁、读锁）
- 排他锁（X锁、写锁）

**按算法**：
- Record Lock（记录锁）
- Gap Lock（间隙锁）
- Next-Key Lock（临键锁）

### 2. 全局锁
```sql
FLUSH TABLES WITH READ LOCK;
```
- 整个数据库只读
- 用于全库备份

### 3. 表级锁
- **表锁**：LOCK TABLES
- **元数据锁（MDL）**：自动加锁，防止 DDL 和 DML 冲突
- **意向锁**：表明事务想要获取行锁
- **AUTO-INC 锁**：自增主键

### 4. 行级锁
- **Record Lock**：锁定单行记录
- **Gap Lock**：锁定记录之间的间隙
- **Next-Key Lock**：Record Lock + Gap Lock

### 5. 间隙锁的作用
- 防止幻读
- 锁定记录之间的间隙
- 只在 REPEATABLE READ 级别生效

### 6. Next-Key Lock
- 左开右闭区间
- 防止幻读
- 可能退化为 Record Lock 或 Gap Lock

### 7. 死锁
**产生条件**：
- 互斥
- 持有并等待
- 不可剥夺
- 循环等待

**解决方法**：
- 超时回滚
- 死锁检测
- 按固定顺序获取锁

### 8. 如何避免死锁
- 合理设计索引
- 尽量使用主键或唯一索引
- 控制事务大小
- 降低隔离级别
- 按相同顺序访问资源

### 9. 乐观锁和悲观锁
**悲观锁**：
- 认为会发生冲突
- 先加锁再操作
- SELECT ... FOR UPDATE

**乐观锁**：
- 认为不会发生冲突
- 使用版本号或时间戳
- 更新时检查版本

```sql
UPDATE table SET value = 2, version = version + 1
WHERE id = 1 AND version = 1;
```

## 五、查询优化

### 1. EXPLAIN 分析
**重要字段**：
- **type**：访问类型（system > const > eq_ref > ref > range > index > ALL）
- **key**：实际使用的索引
- **rows**：扫描的行数
- **Extra**：额外信息

### 2. type 类型
- **system**：表只有一行
- **const**：主键或唯一索引等值查询
- **eq_ref**：主键或唯一索引连接
- **ref**：非唯一索引等值查询
- **range**：范围查询
- **index**：索引全扫描
- **ALL**：全表扫描

### 3. Extra 信息
- **Using index**：覆盖索引
- **Using where**：使用 WHERE 过滤
- **Using temporary**：使用临时表
- **Using filesort**：文件排序
- **Using index condition**：索引下推

### 4. 慢查询优化步骤
1. 开启慢查询日志
2. 使用 EXPLAIN 分析
3. 检查索引使用情况
4. 优化 SQL 语句
5. 调整索引
6. 考虑分库分表

### 5. SQL 优化技巧
- 避免 SELECT *
- 使用 LIMIT 限制返回行数
- 避免在 WHERE 中使用函数
- 使用 JOIN 代替子查询
- 使用 UNION ALL 代替 UNION
- 批量插入代替单条插入
- 使用合适的数据类型

### 6. 分页优化
**传统分页**：
```sql
SELECT * FROM table LIMIT 1000000, 10;
```
- 需要扫描前面的所有行

**优化方案**：
```sql
-- 使用主键
SELECT * FROM table WHERE id > 1000000 LIMIT 10;

-- 使用子查询
SELECT * FROM table WHERE id >=
(SELECT id FROM table LIMIT 1000000, 1) LIMIT 10;
```

### 7. COUNT 优化
- COUNT(*) 和 COUNT(1) 性能相同
- COUNT(列) 不统计 NULL
- MyISAM 存储了行数，COUNT(*) 很快
- InnoDB 需要扫描，可以使用近似值或缓存

### 8. JOIN 优化
- 小表驱动大表
- 使用索引
- 避免笛卡尔积
- 考虑使用 STRAIGHT_JOIN 固定连接顺序

### 9. 子查询优化
- 尽量使用 JOIN 代替
- 使用 EXISTS 代替 IN
- 避免在 WHERE 中使用子查询

### 10. ORDER BY 优化
- 使用索引排序
- 避免 Using filesort
- 利用覆盖索引
- 增大 sort_buffer_size

### 11. GROUP BY 优化
- 使用索引
- 避免临时表
- 使用 WHERE 代替 HAVING

## 六、主从复制

### 1. 主从复制原理
1. 主库写入 binlog
2. 从库 IO 线程读取 binlog 写入 relay log
3. 从库 SQL 线程执行 relay log

### 2. 主从复制的方式
- **异步复制**：主库不等待从库确认
- **半同步复制**：至少一个从库确认
- **全同步复制**：所有从库确认

### 3. 主从延迟的原因
- 从库性能差
- 大事务
- 从库压力大
- 网络延迟
- 单线程复制

### 4. 如何减少主从延迟
- 使用并行复制
- 优化从库性能
- 拆分大事务
- 使用半同步复制
- 读写分离时读主库

### 5. 主从切换
- 手动切换
- 使用 MHA、Orchestrator 等工具
- 使用 MySQL Group Replication

## 七、分库分表

### 1. 为什么要分库分表
- 单表数据量过大
- 并发压力大
- 提高查询性能
- 提高写入性能

### 2. 垂直拆分和水平拆分
**垂直拆分**：
- 按业务拆分
- 不同表放到不同库

**水平拆分**：
- 按数据拆分
- 同一表的数据分散到多个库

### 3. 分片策略
- **范围分片**：按 ID 范围
- **哈希分片**：按 ID 取模
- **地理位置分片**：按地区
- **时间分片**：按时间

### 4. 分库分表带来的问题
- 跨库 JOIN
- 分布式事务
- 全局唯一 ID
- 分页查询
- 排序问题

### 5. 全局唯一 ID 生成方案
- **UUID**：无序，占用空间大
- **数据库自增**：单点问题
- **Redis**：性能好，依赖 Redis
- **雪花算法**：有序，性能好
- **美团 Leaf**：号段模式

### 6. 雪花算法
- 64位 Long 类型
- 1位符号位 + 41位时间戳 + 10位机器ID + 12位序列号
- 有序、高性能
- 依赖时钟

## 八、性能优化

### 1. MySQL 参数优化
**连接相关**：
- max_connections：最大连接数
- max_connect_errors：最大错误连接数

**缓冲相关**：
- innodb_buffer_pool_size：缓冲池大小（建议物理内存的70-80%）
- innodb_log_buffer_size：日志缓冲大小

**日志相关**：
- innodb_flush_log_at_trx_commit：日志刷盘策略
- sync_binlog：binlog 刷盘策略

### 2. innodb_flush_log_at_trx_commit
- **0**：每秒刷盘，性能最好，可能丢失1秒数据
- **1**：每次事务提交刷盘，最安全，性能最差
- **2**：每次提交写入 OS 缓存，每秒刷盘

### 3. sync_binlog
- **0**：由操作系统决定何时刷盘
- **1**：每次事务提交刷盘
- **N**：每 N 次事务提交刷盘

### 4. 表设计优化
- 选择合适的数据类型
- 字段设置 NOT NULL
- 使用 ENUM 代替字符串
- 合理使用 TEXT、BLOB
- 避免过多的列
- 合理使用冗余字段

### 5. 硬件优化
- 使用 SSD
- 增加内存
- 使用多核 CPU
- 使用高速网络

### 6. 架构优化
- 读写分离
- 分库分表
- 使用缓存（Redis）
- 使用消息队列
- 使用 CDN

## 九、备份恢复

### 1. 备份方式
**逻辑备份**：
- mysqldump
- SELECT INTO OUTFILE
- 可读性好，恢复慢

**物理备份**：
- 直接复制数据文件
- XtraBackup
- 恢复快，不可读

### 2. mysqldump
```bash
# 备份
mysqldump -u root -p database > backup.sql

# 恢复
mysql -u root -p database < backup.sql
```

### 3. XtraBackup
- 热备份
- 不锁表
- 增量备份
- 恢复快

### 4. 备份策略
- 全量备份 + 增量备份
- 定期备份
- 异地备份
- 测试恢复

## 十、高可用方案

### 1. 主从复制
- 一主多从
- 读写分离
- 故障切换

### 2. MHA
- Master High Availability
- 自动故障切换
- 数据补偿

### 3. MySQL Group Replication
- 多主模式
- 自动故障检测
- 自动故障转移

### 4. MySQL Cluster
- 分布式数据库
- 高可用
- 高性能

### 5. 双主复制
- 互为主从
- 需要解决主键冲突
- 需要解决数据一致性

## 十一、安全

### 1. 权限管理
```sql
-- 创建用户
CREATE USER 'user'@'host' IDENTIFIED BY 'password';

-- 授权
GRANT SELECT, INSERT ON database.* TO 'user'@'host';

-- 撤销权限
REVOKE INSERT ON database.* FROM 'user'@'host';

-- 删除用户
DROP USER 'user'@'host';
```

### 2. SQL 注入防护
- 使用预编译语句
- 参数化查询
- 输入验证
- 最小权限原则

### 3. 数据加密
- 传输加密（SSL/TLS）
- 存储加密
- 备份加密

## 十二、常见问题

### 1. 为什么 InnoDB 表必须有主键
- 聚簇索引需要主键
- 没有主键会自动创建隐藏主键
- 影响性能

### 2. 自增主键用完了怎么办
- INT：42亿
- BIGINT：9223372036854775807
- 实际很难用完
- 可以修改为 BIGINT

### 3. 为什么不建议使用外键
- 影响性能
- 增加复杂度
- 分库分表困难
- 建议在应用层维护

### 4. 大表如何添加字段
- 使用 pt-online-schema-change
- 使用 gh-ost
- 业务低峰期操作
- 先在从库操作

### 5. 如何删除大量数据
- 分批删除
- 使用 LIMIT
- 避免长事务
- 考虑使用分区表

### 6. 如何优化 INSERT 性能
- 批量插入
- 使用 LOAD DATA INFILE
- 关闭自动提交
- 禁用索引后再启用
- 使用 INSERT DELAYED

### 7. 如何处理表碎片
```sql
-- 查看碎片
SHOW TABLE STATUS LIKE 'table_name';

-- 整理碎片
OPTIMIZE TABLE table_name;
ALTER TABLE table_name ENGINE=InnoDB;
```

### 8. 如何监控 MySQL
- 慢查询日志
- 错误日志
- 二进制日志
- 性能监控（Prometheus + Grafana）
- 使用 pt-query-digest 分析慢查询

## 十三、MySQL 8.0 新特性

### 1. 窗口函数
```sql
SELECT
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as rank
FROM employees;
```

### 2. CTE（公共表表达式）
```sql
WITH cte AS (
    SELECT * FROM table WHERE condition
)
SELECT * FROM cte;
```

### 3. 隐藏索引
```sql
ALTER TABLE table ALTER INDEX index_name INVISIBLE;
```

### 4. 降序索引
```sql
CREATE INDEX idx ON table (col1 ASC, col2 DESC);
```

### 5. JSON 增强
- JSON_TABLE
- JSON_ARRAYAGG
- JSON_OBJECTAGG

### 6. 原子 DDL
- DDL 操作支持事务
- 要么全部成功，要么全部失败

### 7. 默认字符集 utf8mb4
- 支持 emoji
- 完整的 UTF-8 支持

## 十四、实战经验

### 1. 线上问题排查
1. 查看慢查询日志
2. 使用 EXPLAIN 分析
3. 查看锁等待
4. 检查连接数
5. 查看错误日志

### 2. 性能突然下降排查
- 检查是否有慢查询
- 检查是否有锁等待
- 检查缓冲池命中率
- 检查磁盘 IO
- 检查是否有大事务

### 3. 连接数过多处理
- 增加 max_connections
- 使用连接池
- 检查是否有连接泄漏
- 优化慢查询

### 4. 磁盘空间不足处理
- 清理 binlog
- 清理慢查询日志
- 清理错误日志
- 删除无用数据
- 归档历史数据

### 5. 主从延迟处理
- 使用并行复制
- 优化慢查询
- 拆分大事务
- 升级从库硬件

## 十五、面试高频问题

### 1. 如何设计一个高并发系统的数据库
- 读写分离
- 分库分表
- 使用缓存
- 异步处理
- 限流降级

### 2. 如何保证缓存和数据库一致性
- Cache Aside 模式
- 先更新数据库，再删除缓存
- 使用消息队列
- 设置过期时间

### 3. 如何设计一个秒杀系统
- 前端限流
- 使用缓存
- 消息队列异步处理
- 数据库优化
- 库存预扣

### 4. 千万级数据如何分页
- 使用游标
- 使用主键范围
- 避免深分页
- 使用搜索引擎

### 5. 如何实现分布式事务
- 2PC（两阶段提交）
- 3PC（三阶段提交）
- TCC（Try-Confirm-Cancel）
- 本地消息表
- MQ 事务消息
- Saga

### 6. 数据库连接池的作用
- 减少连接创建开销
- 复用连接
- 控制连接数
- 提高性能

### 7. 如何设计一个分布式 ID
- 雪花算法
- 数据库号段
- Redis
- UUID
- 美团 Leaf

### 8. 如何优化千万级数据的查询
- 建立合适的索引
- 分库分表
- 使用缓存
- 读写分离
- 归档历史数据

### 9. 如何保证数据不丢失
- 主从复制
- 半同步复制
- 定期备份
- binlog
- redo log

### 10. 如何处理热点数据
- 使用缓存
- 使用本地缓存
- 数据分片
- 限流

这份 MySQL 八股文涵盖了从基础到高级的所有重要知识点，包括索引、事务、锁、优化、主从复制、分库分表等核心内容，以及实战经验和高频面试题。
