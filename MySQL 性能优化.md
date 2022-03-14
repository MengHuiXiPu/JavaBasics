#  MySQL 性能优化

## MySQL 性能优化总结

既然谈到优化，一定想到要从多个维度进行优化。

这里的优化维度有四个：SQL语句及索引、表结构设计、系统配置、硬件配置。

其中 SQL 语句相关的优化手段是最为重要的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/mngWTkJEOYKht9kicH7icGHOvAiav2hRhq8MibicGrGfWJpcor97qgCOLyuvAu2ANltKtRiaklLOaxCblM81od69f1Eg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 硬件配置

硬件方面的优化可以有 **对磁盘进行扩容、将机械硬盘换为SSD** 等等。但这个优化手段成本最高，见效也较小。

### 系统配置

#### 系统选择

系统通常使用Linux作为服务端的系统，本地开发的话可以随意。Linux 系统版本和 MySQL 版本选择稳定的版本即可。

#### 保证从内存读取

MySQL 会在内存中保存一定的数据，通过 **LRU（最近最少使用）算法**将不常访问的数据保存在硬盘文件中。尽可能的扩大内存中的数据量，将数据保存在内存中，从内存中读取数据，可以提升 MySQL 性能。

MySQL 使用优化过后的 LRU 算法：

> 普通LRU：末尾淘汰法，新数据从链表头部加入，释放空间时从末尾淘汰
>
> 改进LRU：链表分为new和old两个部分，加入元素时并不是从表头插入，而是从中间 midpoint位置插入，如果数据很快被访问，那么page就会向new列表头部移动，如果 数据没有被访问，会逐步向old尾部移动，等待淘汰。每当有新的page数据读取到buffer pool时，InnoDb引擎会判断是否有空闲页，是否足够，如果有就将free page从free list列表删除，放入到LRU列表中。没有空闲页，就会根据LRU算法淘汰LRU链表默认的页，将内存空间释放分配给新的页。

LRU 算法针对的是 MySQL 内存中的结构，这里有个区域叫 **Buffer Pool（缓冲池）** 作为数据读写的缓冲区域。把这个区域进行相应的扩大即可提升性能，当然这个参数要针对服务器硬件的实际情况进行调整。

通过以下命令可以查看相应的BufferPool的相关参数：

```
show global status like 'innodb_buffer_pool_pages_%'
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

输入以下命令可以查看 BufferPool 的大小：

```
show variables like "%innodb_buffer_pool_size%"
```

在这里我们可以修改这个参数的值，如果该服务器是 MySQL 专用的服务器，我们可以 **修改为总内存的 60%~80%** ，当然不能影响系统程序的运行。

这个参数是只读的，可以在 MySQL 的配置文件（my.cnf 或 my.ini）中进行修改。Linux 的配置文件为 **my.cnf**。

```
# 修改缓冲池大小为750M
innodb_buffer_pool_size = 750M
```

#### 数据预热

数据预热相当于将磁盘中的数据提前放入 BufferPool 内存缓冲池内。一定程度提升了读取速度。

对于 InnoDB，这里提供一份预热 SQL 脚本：

```
#mysql5.7版本中，如果DISTINCT和order by一起使用将会报3065错误，sql语句无法执行。这是由于5.7版本语法比之前版本语法要求更加严格导致的。
#推荐在mysql的配置文件my.cnf文件(linux)/my.ini文件(window) 的mysqld中增加或者修改sql_model配置选项
#sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
#重启后生效
SELECT DISTINCT
    CONCAT('SELECT ',rowlist,' FROM ',db,'.',tb,
    ' ORDER BY ',rowlist,';') selectSql
    FROM
    (
        SELECT
            engine,table_schema db,table_name tb,
            index_name,GROUP_CONCAT(column_name ORDER BY seq_in_index) rowlist
        FROM
        (
            SELECT
                B.engine,A.table_schema,A.table_name,
                A.index_name,A.column_name,A.seq_in_index
            FROM
                information_schema.statistics A INNER JOIN
                (
                    SELECT engine,table_schema,table_name
                    FROM information_schema.tables WHERE
                    engine='InnoDB'
                ) B USING (table_schema,table_name)
            WHERE B.table_schema NOT IN ('information_schema','mysql')
            ORDER BY table_schema,table_name,index_name,seq_in_index
        ) A
        GROUP BY table_schema,table_name,index_name
    ) AA 
ORDER BY db,tb;
```

#### 降低磁盘的写入次数

（1）增大 redo log，减少落盘次数：

redo log 是重做日志，用于保证数据的一致，减少落盘相当于减少了系统 IO 操作。

innodb_log_file_size 设置为 0.25 * innodb_buffer_pool_size

（2）通用查询日志、慢查询日志可以不开 ，binlog 可开启。

通用查询和慢查询日志也是要落盘的，可以根据实际情况开启，如果不需要使用的话就可以关掉。binlog 用于恢复和主从复制，这个可以开启。

查看相关参数的命令：

```
# 慢查询日志
show variables like 'slow_query_log%'
# 通用查询日志
show variables like '%general%';
# 错误日志
show variables like '%log_error%'
# 二进制日志
show variables like '%binlog%';
```

（3）写 redo log 策略 innodb_flush_log_at_trx_commit 设置为 0 或 2

对于不需要强一致性的业务，可以设置为 0 或 2。

- 0：每隔 1 秒写日志文件和刷盘操作（写日志文件 LogBuffer --> OS cache，刷盘 OS cache --> 磁盘文件），最多丢失 1 秒数据
- 1：事务提交，立刻写日志文件和刷盘，数据不丢失，但是会频繁 IO 操作
- 2：事务提交，立刻写日志文件，每隔 1 秒钟进行刷盘操作

#### 系统调优参数

**back_log**

back_log值可以指出在MySQL暂时停止回答新请求之前的短时间内多少个请求可以被存在堆栈中。也就是说，如果MySQL的连接数据达到max_connections时，新来的请求将会被存在堆栈中，以等待某一连接释放资源，该堆栈的数量即back_log，如果等待连接的数量超过back_log，将不被授予连接资源。可以从默认的50升至500。

**wait_timeout**

数据库连接闲置时间，闲置连接会占用内存资源。可以从默认的8小时减到半小时。

**max_user_connection**

最大连接数，默认为0无上限，最好设一个合理上限。

**thread_concurrency**

并发线程数，设为CPU核数的两倍。

**skip_name_resolve**

禁止对外部连接进行DNS解析，消除DNS解析时间，但需要所有远程主机用IP访问。

**key_buffer_size**

索引块的缓存大小，增加会提升索引处理速度，对MyISAM表性能影响最大。对于内存4G左右，可设为256M或384M，通过查询show status like 'key_read%'，保证key_reads / key_read_requests在0.1%以下最好。

**innodb_buffer_pool_size**

缓存数据块和索引块，对InnoDB表性能影响最大。通过查询show status like 'Innodb_buffer_pool_read%'，保证 (Innodb_buffer_pool_read_requests – Innodb_buffer_pool_reads) / Innodb_buffer_pool_read_requests越高越好。

**innodb_additional_mem_pool_size**

InnoDB存储引擎用来存放数据字典信息以及一些内部数据结构的内存空间大小，当数据库对象非常多的时候，适当调整该参数的大小以确保所有数据都能存放在内存中提高访问效率，当过小的时候，MySQL会记录Warning信息到数据库的错误日志中，这时就需要该调整这个参数大小。

**innodb_log_buffer_size**

InnoDB存储引擎的事务日志所使用的缓冲区，一般来说不建议超过32MB。

**query_cache_size**

缓存MySQL中的ResultSet，也就是一条SQL语句执行的结果集，所以仅仅只能针对select语句。当某个表的数据有任何变化，都会导致所有引用了该表的select语句在Query Cache中的缓存数据失效。所以，当我们数据变化非常频繁的情况下，使用Query Cache可能得不偿失。根据命中率(Qcache_hits/(Qcache_hits+Qcache_inserts)*100))进行调整，一般不建议太大，256MB可能已经差不多了，大型的配置型静态数据可适当调大。可以通过命令show status like 'Qcache_%'查看目前系统Query catch使用大小。

**read_buffer_size**

MySQL读入缓冲区大小。对表进行顺序扫描的请求将分配一个读入缓冲区，MySQL会为它分配一段内存缓冲区。如果对表的顺序扫描请求非常频繁，可以通过增加该变量值以及内存缓冲区大小来提高其性能。

**sort_buffer_size**

MySQL执行排序使用的缓冲大小。如果想要增加ORDER BY的速度，首先看是否可以让MySQL使用索引而不是额外的排序阶段。如果不能，可以尝试增加sort_buffer_size变量的大小。

**read_rnd_buffer_size**

MySQL的随机读缓冲区大小。当按任意顺序读取行时(例如按照排序顺序)，将分配一个随机读缓存区。进行排序查询时，MySQL会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速度，如果需要排序大量数据，可适当调高该值。但MySQL会为每个客户连接发放该缓冲空间，所以应尽量适当设置该值，以避免内存开销过大。

**record_buffer**

每个进行一个顺序扫描的线程为其扫描的每张表分配这个大小的一个缓冲区。如果你做很多顺序扫描，可能想要增加该值。

**thread_cache_size**

保存当前没有与连接关联但是准备为后面新的连接服务的线程，可以快速响应连接的线程请求而无需创建新的。

**table_cache**

类似于thread_cache _size，但用来缓存表文件，对InnoDB效果不大，主要用于MyISAM。

### 表结构设计

#### 设计中间表

设计中间表，一般针对于统计分析功能，或者实时性不高的需求（报表统计，数据分析等系统）。

#### 设计冗余字段

为减少关联查询，创建合理的冗余字段（创建冗余字段还需要注意数据一致性问题）。这里分库分表时较为常用。

#### 拆表

对于字段太多的大表，考虑拆表（比如一个表有100多个字段） 对于表中经常不被使用的字段或者存储数据比较多的字段，考虑拆表。

#### 主键优化

每张表建议都要有一个主键（主键索引），而且主键类型最好是int类型，建议自增主键（分布式系统的情况下建议雪花算法）

#### 字段的设计

数据库中的表越小，在它上面执行的查询也就会越快。因此，在创建表的时候，为了获得更好的性能，我们可以将表中字段的宽度设得尽可能小。

- 使用可以存下数据最小的数据类型，合适即可
- 尽量使用TINYINT、SMALLINT、MEDIUM_INT作为整数类型而非INT，如果非负则加上UNSIGNED；
- VARCHAR的长度只分配真正需要的空间；
- 对于某些文本字段，比如"省份"或者"性别"，使用枚举或整数代替字符串类型；在MySQL中， ENUM类型被当作数值型数据来处理，而数值型数据被处理起来的速度要比文本类型快得多
- 尽量使用TIMESTAMP而非DATETIME；
- 单表不要有太多字段，建议在20以内；
- 尽可能使用 not null 定义字段，null 占用4字节空间，这样在将来执行查询的时候，数据库不用去比较NULL值。
- 用整型来存IP。
- 尽量少用 text 类型，非用不可时最好考虑拆表

### MySQL语句及索引

如果发现SQL查询比较慢，可以开启慢查询日志进行排查。

```
# 开启全局慢查询日志
SET global slow_query_log = ON;
# 设置慢查询日志文件名
SET global slow_query_log_file = 'slow-query.log';
# 记录未使用索引的SQL
SET global log_queries_not_using_indexes = ON;
# 慢查询的时间阈值，默认10秒
SET long_query_time = 10;
```

注：索引并不是越多越好，要根据查询有针对性的创建。

#### 索引创建和使用原则

- 单表查询：哪个列作查询条件，就在该列创建索引
- 多表查询：left join 时，索引添加到右表关联字段；right join 时，索引添加到左表关联字段
- 不要对索引列进行任何操作（计算、函数、类型转换）
- 索引列中不要使用 !=，<> 非等于
- 字符字段只建前缀索引，最好不要做主键；
- 尽量不用UNIQUE，由程序保证约束
- 不用外键，由程序保证约束
- 索引列不要为空，且不要使用 is null 或 is not null 判断
- 索引字段是字符串类型，查询条件的值要加''单引号，避免底层类型自动转换

#### 使用 EXPLAIN 分析 SQL

这里对explain的结果进行简单说明：

- select_type：查询类型

- - SIMPLE 简单查询
  - PRIMARY 最外层查询
  - UNION union后续查询
  - SUBQUERY 子查询

- type：查询数据时采用的方式

- - ALL 全表**（性能最差）**
  - index 基于索引的全表
  - range 范围 （< > in）
  - ref 非唯一索引单值查询
  - const 使用主键或者唯一索引等值查询

- possible_keys：可能用到的索引

- key：真正用到的索引

- rows：预估扫描多少行记录

- key_len：使用了索引的字节数

- Extra：额外信息

- - Using where 索引回表
  - Using index 索引直接满足条件
  - Using filesort 需要排序
  - Using temprorary 使用到临时表

对于以上的几个列，我们重点关注的是type，最直观的反映出SQL的性能。

#### SQL语句尽可能简单

一条sql只能在一个cpu运算；大语句拆小语句，减少锁时间；一条大sql可以堵死整个库。

#### 对于连续数值，使用 BETWEEN 不用 IN

SELECT id FROM t WHERE num BETWEEN 1 AND 5；

#### SQL 语句中 IN 包含的值不应过多

MySQL对于IN做了相应的优化，即将IN中的常量全部存储在一个数组里面，而且这个数组是排好序的。如果数值较多，需要在内存进行排序操作，产生的消耗也是比较大的。

#### SELECT 语句必须指明字段名称

SELECT * 增加很多不必要的消耗（CPU、IO、内存、网络带宽）；减少了使用覆盖索引的可能性。

##### 当只需要一条数据的时候，使用 limit 1

limit 相当于截断查询。

例如：对于select * from user limit 1; 虽然进行了全表扫描，但是limit截断了全表扫描，从0开始取了1条数据。

#### 排序字段加索引

排序的字段建立索引在排序的时候也会用到

#### 如果限制条件中其他字段没有索引，尽量少用or

#### 尽量用 union all 代替 union

union和union all的差别就在于union会对数据做一个distinct的动作，而这个distanct动作的速度则取决于现有数据的数量，数量越大则时间也越慢。而对于几个数据集，要确保数据集之间的数据互相不重复，基本是O(n)的算法复杂度。

#### 区分 in 和 exists、not in 和 not exists

如果是exists，那么以外层表为驱动表，先被访问，如果是IN，那么先执行子查询。所以IN适合于外表大而内表小的情况；EXISTS适合于外表小而内表大的情况。

#### 使用合理的分页方式以提高分页的效率

limit m n，其中的m偏移量尽量小。m越大查询越慢。

#### 避免使用 % 前缀模糊查询

例如：like '%name'或者like '%name%'，这种查询会导致索引失效而进行全表扫描。但是可以使用like 'name%'，这种会使用到索引。

#### 避免在 where 子句中对字段进行表达式操作

这种不会使用到索引：

```
select user_id,user_project from user_base where age*2=36;
```

可以改为：

```
select user_id,user_project from user_base where age=36/2;
```

任何对列的操作都将导致表扫描，它包括数据库函数、计算表达式等等，查询时要尽可能将操作移至等号右边。

#### 避免隐式类型转换

where 子句中出现的 column 字段要和数据库中的字段类型对应

#### 必要时可以使用 force index 来强制查询走某个索引

有的时候 MySQL 优化器采取它认为合适的索引来检索 SQL 语句，但是可能它所采用的索引并不是我们想要的。这时就可以采用 forceindex 来强制优化器使用我们制定的索引。

#### 使用联合索引时注意范围查询

对于联合索引来说，如果存在范围查询，比如between、>、<等条件时，会造成后面的索引字段失效。

#### 某些情况下，可以使用连接代替子查询

因为使用 join，MySQL 不会在内存中创建临时表。

#### 使用JOIN的优化

使用小表驱动大表，例如使用inner join时，优化器会选择小表作为驱动表

#### 小表驱动大表，即小的数据集驱动大的数据集

如：以 A，B 两表为例，两表通过 id 字段进行关联。

```
#当 B 表的数据集小于 A 表时，用 in 优化 exist；使用 in ，两表执行顺序是先查 B 表，再查 A 表
select * from A where id in (select id from B)

#当 A 表的数据集小于 B 表时，用 exist 优化 in；使用 exists，两表执行顺序是先查 A 表，再查 B 表
select * from A where exists (select 1 from B where B.id = A.id)
```





------


上面都是一些常规的优化方法，我们还可以使用：读写分离、表分区、分库分表等。但读写分离还好，分库分表涉及到的架构变化太大了，考虑的东西太多了，不到迫不得已的情况下，尽量不要采用。

你学会了么？