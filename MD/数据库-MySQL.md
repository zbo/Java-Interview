
## MySQL
### 引擎对比
1. InnoDB支持事务
2. InnoDB支持外键
3. InnoDB有行级锁，MyISAM是表级锁

MyISAM相对简单所以在效率上要优于InnoDB。如果系统插入和查询操作多，不需要事务外键。选择MyISAM  
如果需要频繁的更新、删除操作，或者需要事务、外键、行级锁的时候。选择InnoDB。

### [数据库性能优化](https://www.zhihu.com/question/19719997)
1. 优化SQL语句和索引，在where/group by/order by中用到的字段建立索引，索引字段越小越好，复合索引建立的顺序
2. 加缓存，Memcached, Redis
3. 主从复制，读写分离
4. 垂直拆分，其实就是根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统
5. 水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key,为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；

### [SQL优化](https://www.imooc.com/video/3711)：
优化步骤：

1. 根据慢日志定位慢查询SQL
2. 用explain分析SQL（type和extra字段分析）
3. 修改SQL或加索引（如下）

* 对经常查询的列建立索引，但索引建多了当数据改变时修改索引会增大开销
* 使用精确列名查询而不是*，特别是当数据量大的时候
* 减少[子查询](http://www.cnblogs.com/zhengyun_ustc/p/slowquery3.html)，使用Join替代
* 不用NOT IN，因为会使用全表扫描而不是索引；不用IS NULL，NOT IS NULL，因为会使索引、索引统计和值更加复杂，并且需要额外一个字节的存储空间。

问：max(xxx)如何用索引优化？
答：在xxx列上建立索引，因为索引是B+树顺序排列的，在下次查询的时候就会使用索引来查询到最大的值是哪个

问：有个表特别大，字段是姓名、年龄、班级，如果调用`select * from table where name = xxx and age = xxx`该如何通过建立索引的方式优化查询速度？  
答：由于mysql查询每次只能使用一个索引，如果在name、age两列上创建联合索引的话将带来更高的效率。如果我们创建了(name, age)的联合索引，那么其实相当于创建了(name, age)、(name)2个索引。因此我们在创建复合索引时应该将最常用作限制条件的列放在最左边，依次递减。其次还要考虑该列的数据离散程度，如果有很多不同的值的话建议放在左边，name的离散程度也大于age。

问：优化大数据量的分页查询？
1. 索引优化：确保用于排序和筛选的字段已经建立了索引
2. 延迟关联：先获取主键，然后再根据主键获取完整的行数据。这样可以减少数据库在排序时需要处理的数据量。通过将 select * 转变为 select id，把符合条件的 id 筛选出来后，最后通过嵌套查询的方式按顺序取出 id 对应的行
3. 业务限制：查询页数上限
4. 使用其他数据结构如ES、分表

```sql
-- 优化前
select *
from people
order by create_time desc
limit 5000000, 10;

-- 优化后
select a.*
from people a
inner join(
    select id
    from people
    order by create_time desc
    limit 5000000, 10
) b ON a.id = b.id;
```

#### 最左前缀匹配原则
定义：在检索数据时从联合索引的最左边开始匹配。一直向右匹配直到遇到范围查询（>/</between/like）就停止后面的匹配。比如查询a = 3, b = 4 and c > 5 and d = 6，如果建立(a,b,c,d)索引，d是用不到索引的，如果建立(a,b,d,c)索引，则都可以用到，此外a,b,d的顺序可以任意调整

联合索引原理：联合索引是将第一个字段作为非叶子节点形成B+树结构的，在查询索引的时候先根据该字段一步步查询到具体的叶子节点，叶子节点上还会存在第二个字段甚至第三个字段的已排序结果。所以对于(a,b,c,d)这个联合索引会存在(a)，(a,b)，(a,b,c)，(a,b,c,d)四个索引

## 事务隔离级别
1. 原子性（Atomicity）：事务作为一个整体被执行 ，要么全部执行，要么全部不执行；
2. 一致性（Consistency）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性地完成预定的工作；
3. 隔离性（Isolation）：多个事务并发执行时，一个事务的执行不应影响其他事务的执行，然后你可以扯到隔离级别；
4. 持久性（Durability）：一个事务一旦提交，对数据库的修改应该永久保存。

不同事务隔离级别的问题：  
[https://www.jianshu.com/p/4e3edbedb9a8](https://www.jianshu.com/p/4e3edbedb9a8)

## 锁表、锁行
1. InnoDB 支持表锁和行锁，使用索引作为检索条件修改数据时采用行锁，否则采用表锁
2. InnoDB 自动给修改操作加锁，给查询操作不自动加锁
3. 行锁相对于表锁来说，优势在于高并发场景下表现更突出，毕竟锁的粒度小
4. 当表的大部分数据需要被修改，或者是多表复杂关联查询时，建议使用表锁优于行锁

[https://segmentfault.com/a/1190000012773157](https://segmentfault.com/a/1190000012773157)

### 悲观锁乐观锁、如何写对应的SQL
悲观锁：select for update  
乐观锁：先查询一次数据，然后使用查询出来的数据+1进行更新数据，如果失败则循环  

[https://www.jianshu.com/p/f5ff017db62a](https://www.jianshu.com/p/f5ff017db62a)

### MVVC
https://tech.meituan.com/2014/08/20/innodb-lock.html

### 索引
#### 原理
聚集索引：数据行的物理顺序与列值（一般是主键的那一列）的逻辑顺序相同，所以一个表中只能拥有一个聚集索引。叶子结点即存储了真实的数据行，不再有另外单独的数据页  
非聚集索引：该索引中索引的逻辑顺序与磁盘上行的物理存储顺序不同，所以一个表中可以拥有多个非聚集索引。叶子结点包含索引字段值及指向数据页数据行的逻辑指针  

#### 索引原理
使用B+树来构建索引，为什么不用二叉树？因为红黑树在磁盘上的查询性能远不如B+树

红黑树最多只有两个子节点，所以高度会非常高，导致遍历查询的次数会多，又因为红黑树在数组中存储的方式，导致逻辑上很近的父节点与子节点可能在物理上很远，导致无法使用磁盘预读的局部性原理，需要很多次IO才能找到磁盘上的数据

但B+树一个节点中可以存储很多个索引的key，且将大小设置为一个页，一次磁盘IO就能读取很多个key，且叶子节点之间还加上了下个叶子节点的指针，遍历索引也会很快。

B+树的高度如何计算？
1. 单个叶子节点（页）中的记录数 =16K/1K=16。（这里假设一行记录的数据大小为1k）
2. 假设主键 ID 为 bigint 类型，长度为 8 字节，而指针大小在 InnoDB 源码中设置为 6 字节，这样一共 14 字节
3. 我们一个页中能存放多少这样的单元，其实就代表有多少指针，即 16384/14=1170。
4. 那么可以算出一棵高度为 2 的 B+ 树，能存放 1170*16=18720 条这样的数据记录。

B与B+区别:
1. b+树的中间节点不保存数据，所以磁盘页能容纳更多节点元素；
2. b+树查询必须查找到叶子节点，b树只要匹配到即可不用管元素位置，因此b+树查找更稳定
3. 对于范围查找来说，b+树只需遍历叶子节点链表即可，b树却需要重复地中序遍历

#### 优化
如何选择合适的列建立索引？

1. WHERE / GROUP BY / ORDER BY / ON 的列
2. 离散度大（不同的数据多）的列使用索引才有查询效率提升
3. 索引字段越小越好，因为数据库按页存储的，如果每次查询IO读取的页越少查询效率越高

### count1/count*/count字段 的区别

1. count(*)包括了所有的列，相当于行数，在统计结果的时候，不会忽略列值为NULL
2. count(1)包括了忽略所有列，用1代表代码行，在统计结果的时候，不会忽略列值为NULL 
3. count(列名)只包括列名那一列，在统计结果的时候，会忽略列值为空（这里的空不是只空字符串或者0，而是表示null）的计数，即某个字段值为NULL时，不统计。

count(*),自动会优化指定到那一个字段。所以没必要去count(1)，用count(*)，sql会帮你完成优化的 因此：count(1)和count(*)基本没有差别！

## 分区分库分表
分区：把一张表的数据分成N个区块，在逻辑上看最终只是一张表，但底层是由N个物理区块组成的，通过将不同数据按一定规则放到不同的区块中提升表的查询效率。

水平分表：为了解决单标数据量过大（数据量达到千万级别）问题。所以将固定的ID hash之后mod，取若0~N个值，然后将数据划分到不同表中，需要在写入与查询的时候进行ID的路由与统计

垂直分表：为了解决表的宽度问题，同时还能分别优化每张单表的处理能力。所以将表结构根据数据的活跃度拆分成多个表，把不常用的字段单独放到一个表、把大字段单独放到一个表、把经常使用的字段放到一个表

分库：面对高并发的读写访问，当数据库无法承载写操作压力时，不管如何扩展slave服务器，此时都没有意义了。因此数据库进行拆分，从而提高数据库写入能力，这就是分库。

问题：

1. 事务问题。在执行分库之后，由于数据存储到了不同的库上，数据库事务管理出现了困难。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价；如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。
2. 跨库跨表的join问题。在执行了分库分表之后，难以避免会将原本逻辑关联性很强的数据划分到不同的表、不同的库上，我们无法join位于不同分库的表，也无法join分表粒度不同的表，结果原本一次查询能够完成的业务，可能需要多次查询才能完成。
3. 额外的数据管理负担和数据运算压力。额外的数据管理负担，最显而易见的就是数据的定位问题和数据的增删改查的重复执行问题，这些都可以通过应用程序解决，但必然引起额外的逻辑运算。
4. 

## MySQL数据库主从延迟同步方案
什么事主从延迟：为了完成主从复制，从库需要通过 I/O 线程获取主库中 dump 线程读取的 binlog 内容并写入到自己的中继日志 relay log 中，从库的 SQL 线程再读取中继日志，重做中继日志中的日志，相当于再执行一遍 SQL，更新自己的数据库，以达到数据的一致性。主从延迟，就是同一个事务，从库执行完成的时间与主库执行完成的时间之差

1. 「异步复制」：MySQL 默认的复制即是异步的，主库在执行完客户端提交的事务后会立即将结果返给客户端，并不关心从库是否已经接收并处理。这样就会有一个问题，一旦主库宕机，此时主库上已经提交的事务可能因为网络原因并没有传到从库上，如果此时执行故障转移，强行将从提升为主，可能导致新主上的数据不完整。
2. 「全同步复制」：指当主库执行完一个事务，并且所有的从库都执行了该事务，主库才提交事务并返回结果给客户端。因为需要等待所有从库执行完该事务才能返回，所以全同步复制的性能必然会收到严重的影响。
3. 「半同步复制」：是介于全同步复制与全异步复制之间的一种，主库只需要等待至少一个从库接收到并写到 Relay Log 文件即可，主库不需要等待所有从库给主库返回 ACK。主库收到这个 ACK 以后，才能给客户端返回 “事务完成” 的确认。这样会出现一个问题，就是实际上主库已经将该事务 commit 到了存储引擎层，应用已经可以看到数据发生了变化，只是在等待返回而已。如果此时主库宕机，可能从库还没写入 Relay Log，就会发生主从库数据不一致。为了解决上述问题，MySQL 5.7 引入了增强半同步复制。针对上面这个图，“Waiting Slave dump” 被调整到了 “Storage Commit” 之前，即主库写数据到 binlog 后，就开始等待从库的应答 ACK，直到至少一个从库写入 Relay Log 后，并将数据落盘，然后返回给主库 ACK，通知主库可以执行 commit 操作，然后主库再将事务提交到事务引擎层，应用此时才可以看到数据发生了变化。
4. 一主多从：如果从库承担了大量查询请求，那么从库上的查询操作将耗费大量的 CPU 资源，从而影响了同步速度，造成主从延迟。那么我们可以多接几个从库，让这些从库来共同分担读的压力。
5. 强制走主库：如果某些操作对数据的实时性要求比较苛刻，需要反映实时最新的数据，比如说涉及金钱的金融类系统、在线实时系统、又或者是写入之后马上又读的业务，这时我们就得放弃读写分离，让此类的读请求也走主库，这就不存延迟问题了。
6. 并行复制：MySQL在5.6版本中使用并行复制来解决主从延迟问题。所谓并行复制，指的就是从库开启多个SQL线程，并行读取relay log中不同库的日志，然后并行重放不同库的日志，这是库级别的并行。MySQL 5.7对并行复制做了进一步改进，开始支持真正的并行复制功能。MySQL5.7的并行复制是基于组提交的并行复制，官方称为enhanced multi-threaded slave（简称MTS），延迟问题已经得到了极大的改善。其核心思想是：一个组提交的事务都是可以并行回放（配合binary log group commit）；slave机器的relay log中 last_committed相同的事务（sequence_num不同）可以并发执行。

强制走主库之后，如何在处理大数据量的时候保证实时写查：
1. 表结构与索引优化：确保表结构设计合理，尽量避免数据冗余和更新异常。根据查询条件创建合适的索引，特别是对于高频查询字段，可以显著提高读取速度。
2. 分区表（Partitioning）：对于非常大的表，可以考虑使用分区表功能，将大表物理分割成多个小表，根据时间、范围或其他业务逻辑进行划分，这样不仅能提高查询效率，还可以通过并行处理提高写入速度。
3. 分库分表：当单表数据量过大时，采用水平拆分或垂直拆分策略将数据分布到多个数据库或表中，进一步提升读写性能。
4. 查询优化：避免全表扫描，尽可能使用覆盖索引。编写高效的SQL语句，减少不必要的计算和排序操作。
5. 缓存机制：在数据库层面，可以考虑使用内存表或者缓存技术（如Redis）来临时存储最近写入的数据，以便快速响应实时查询请求。
6. 实时分析型数据库：对于大数据量且实时性要求极高的场景，可以考虑引入实时分析型数据库系统，例如ClickHouse、Druid等，它们设计之初就为实时写入和查询提供了高效的支持。

欢迎光临[我的博客](http://www.wangtianyi.top/?utm_source=github&utm_medium=github)，发现更多技术资源~
