---
title: ClickHouse入门（一）
date: 2021-03-24
excerpt_separator: "<!--more-->"
tags:  
   - clickhouse
   - OLAP   
categories: 
   - OLAP  
author: linyuliang  
description: ClickHouse入门教程一
toc: true
toc_sticky: true
---
本文是ClickHouse入门教程（一），目标是学会ClickHouse的基本应用使用，不涉及ClickHouse的安装部署配置运维。以下内容基本从以下参考资料中拷贝而来：

参考资料：
- 《ClickHouse原理解析与应用实践》
- [ClickHouse官方文档](https://clickhouse.tech/docs/zh/)
- [ClickHouse概述](https://www.jianshu.com/p/350b59e8ea68)
- [ClickHouse](https://github.com/leelovejava/doc/blob/4a595f27590948a1f5b705610b95cdfa9902abc2/dataBase/ClickHouse/ClickHouse.md)
- [ClickHouse各种MergeTree的关系与作用](https://mp.weixin.qq.com/s/0bXEfAN3xRXOWgGKr2GQZg)
<!-- more -->
## 什么是Clickhouse（Click Stream, Data Ware-House）
　　ClickHouse是一个开源的用于快速联机分析(OLAP)的列式数据库管理系统(DBMS)。是俄罗斯的Yandex公司于2016年开源的列式存储数据库（DBMS），主要用于在线分析处理查询（OLAP），能够使用SQL查询实时生成分析数据报告。这个列式存储数据库的跑分要超过很多流行的商业MPP数据库软件，例如Vertica。  
## 发展历程 
    摘自：《ClickHouse原理解析与应用实践》
　　ClickHouse背后的研发团队是来自俄罗斯的Yandex公司。这是一家俄罗斯本土的互联网企业，于2011年在纳斯达克上市，它的核心产品是搜索引擎。根据最新的数据显示，Yandex占据了本国47%以上的搜索市场，是现今世界上最大的俄语搜索引擎。Google是它的直接竞争对手。
　　众所周知，在线搜索引擎的营收来源非常依赖流量和在线广告业务。所以，通常搜索引擎公司为了更好地帮助自身及用户分析网络流量，都会推出自家的在线流量分析产品，例如Google的Google Analytics、百度的百度统计。Yandex也不例外，Yandex.Metrica就是这样一款用于在线流量分析的产品（https://metrica.yandex.com）。
　　ClickHouse就是在这样的产品背景下诞生的，伴随着Yandex.Metrica业务的发展，其底层架构历经四个阶段，一步一步最终形成了大家现在所看到的ClickHouse。纵观这四个阶段的发展，俨然是数据分析产品形态以及OLAP架构历史演进的缩影。通过了解这段演进过程，我们能够更透彻地了解OLAP面对的挑战，以及ClickHouse能够解决的问题。
### 顺理成章的MySQL时期（ROLAP 固定报告）
- MyISAM表引擎
### 另辟蹊径的Metrage时期（MOLAP 固定报告）
- key-value模型代替关系模型
- 使用LSM树代替B+树（最具代表性的使用LSM树索引结构的系统是HBase）
- 实时查询改成了预处理方式，将需要分析的数据预先聚合（类似MOLAP）
### 自我突破的OLAPServer时期（HOLAP（Metrage+OLAPServer）自助报告，产品形态升级倒逼，固定化的分析形式已不满足用户的需求，与Metrage互补的定位）
- 自主研发OLAPServer系统，设计成专门处理自定义报告这类临时性分析需求的系统，与Metrage系统形成互补关系
- 从Key-Value模型换回了关系模型，因为关系模型具有更好的描述能力，使用SQL作为查询语言，更大众化
- 存储结构和索引，结合了MyISAM（关注写入和插入，不支持事务）和LSM树（稀疏索引）最精华的部分
- 引入了列式存储，将索引文件和数据文件按照列字段的粒度进行了拆分，每个列字段各自独立存储，减少数据读取范围
- 没有DBMS应有的基本管理功能（DDL查询等）
### 水到渠成的ClickHouse时代（ROLAP 自助报告 实时聚合）
- MOLAP（预先聚合）的问题：
  - 无法满足自定义分析的需求，只能支持固定的分析场景
  - 维度组合爆炸会导致数据膨胀，造成不必要的计算和存储开销
  - 流量数据是在线实时接收的，所以预聚合还需要考虑如何及时更新数据
- 以OLAPServer为基础进一步完善，以实现一个完备的数据库管理系统（DBMS）为目标，最终打造出了ClickHouse，并于2016年开源。
## 适用场景
　　ClickHouse非常适用于商业智能领域（BI领域），可以应用于以下场景：
1. 电信行业用于存储数据和统计数据使用。
2. 新浪微博用于用户行为数据记录和分析工作。
3. 用于广告网络和RTB,电子商务的用户行为分析。
4. 信息安全里面的日志分析。
5. 检测和遥感信息的挖掘。
6. 网络游戏以及物联网的数据处理和价值数据分析。
7. 最大的应用来自于Yandex的统计分析服务Yandex.Metrica，类似于谷歌Analytics(GA)，或友盟统计，小米统计，帮助网站或移动应用进行数据分析和精细化运营工具，据称Yandex.Metrica为世界上第二大的网站分析平台。ClickHouse在这个应用中，部署了近四百台机器，每天支持200亿的事件和历史总记录超过13万亿条记录，这些记录都存有原始数据（非聚合数据），随时可以使用SQL查询和分析，生成用户报告。
　　ClickHouse作为一款高性能OLAP数据库，虽然很优秀，但是也不是万能的。我们不应该把它用于任何OLTP事务性操作的场景，它有以下几点不足：
1. 不支持事务
2. 不擅长根据主键按行粒度进行查询，更新和删除操作（虽然支持）
3. 不支持高并发，官方建议qps为100，可以通过修改配置文件增加连接数，但是在服务器足够好的情况下
## 重要特性：  
- 真正的列式数据库管理系统,完备的DBMS功能
  - DDL（数据定义语言）：可以动态地创建、修改或删除数据库、表和视图，而无须重启服务。
  - DML（数据操作语言）：可以动态查询、插入、修改或删除数据。
  - 权限控制：可以按照用户粒度设置数据库或者表的操作权限，保障数据的安全性。
  - 数据备份与恢复：提供了数据备份导出与导入恢复机制，满足生产环境的要求。
  - 分布式管理：提供集群模式，能够自动管理多个数据库节点。
- 列式存储与数据压缩
  - 列式存储除了降低IO和存储的压力外，还为向量化执行做好了铺垫
- 向量化执行引擎
  - 在CPU寄存器层面实现数据的并行操作（）
- 关系模型与SQL查询：在许多情况下与ANSI SQL标准相同
  - ClickHouse是大小写敏感的
- 多样化的表引擎
  - 每种引擎有自己的特点，以满足各类实际业务场景的要求
- 多线程与分布式
  - 软件层面，大量使用了多线程技术提速
  - 支持分区（纵向扩展，利用多线程原理），分片（横向扩展，利用分布式原理）
- 多主架构
  - 适合用于多数据中心，异地多活的场景
- 适合在线查询
- 数据分片与分布式查询
  - 分片的数量取决于节点数量（1个分片只能对应1个服务节点）
- 不依赖Hadoop复杂生态
## ClickHouse为什么这么快
　　ClickHouse的设计采用了自下而上的方式。
1. 着眼硬件，先想后做（在什么样的硬件上，实现什么样的性能，用什么数据结构，怎么工作）
2. 算法在前，抽象在后（性能是算法选择的首要考量指标）
3. 用于尝鲜，不行就换（使用最合适，最快的算法）
4. 特定场景，特殊优化（同一个场景的不同状况，选择使用不同的实现方式，尽可能将性能最大化）
5. 持续测试，持续改进  

## [ClickHouse的数据类型](https://clickhouse.tech/docs/zh/sql-reference/data-types/)
### 基础类型
1. 数值类型
   1. Int
    固定长度的整型，包括有符号整型或无符号整型。

    - 整型范围
      
      | 类型 | 字节 | 范围 | 普遍观念 |
      | -------- | -------- | -------- | -------- |
      | Int8 | 1 | -128:127 | Tinyint |
      | Int16 | 2 | -32768:32767 | SmallInt |
      | Int32 | 4 | -2147483648:2147483647 | Int |
      | Int64 | 8 | -9223372036854775808:9223372036854775807 | Bigint |
    - 无符号整型范围
      
      | 类型 | 字节 | 范围 | 普遍观念 |
      | -------- | -------- | -------- | -------- |
      | UInt8 | 1 | 0:255 | Tinyint Unsigned |
      | UInt16 | 2 | 0:65535 | SmallInt Unsigned |
      | UInt32 | 4 | 0:4294967295 | Int Unsigned |
      | UInt64 | 8 | 0:18446744073709551615 | Bigint Unsigned |
  
   2. Float
    - 单精度浮点数  
       Float32从小数点后第8位起会发生数据溢出  
      
      | 类型 | 字节 | 有效精度（位数）| 普遍概念 |  
      | -------- | -------- | -------- | -------- |  
      | Float32 | 4 | 7 | Float |
      
     - 双精度浮点数  
       Float32从小数点后第17位起会发生数据溢出  
       
       | 类型 | 字节 | 有效精度（位数）| 普遍概念 | 
       | -------- | -------- | -------- | --------  |
       | Float64 | 8 | 16 | Double |
  
   3. Decimal
    有符号的定点数，可在加、减和乘法运算过程中保持精度。ClickHouse提供了Decimal32、Decimal64和Decimal128三种精度的定点数，支持几种写法：
    - Decimal(P, S)  
    - Decimal32(S)  
    数据范围：( -1 * 10^(9 - S), 1 * 10^(9 - S) )
    - Decimal64(S)  
    数据范围：( -1 * 10^(18 - S), 1 * 10^(18 - S) )
    - Decimal128(S)  
    数据范围： ( -1 * 10^(38 - S), 1 * 10^(38 - S) )
    - Decimal256(S)  
    数据范围：( -1 * 10^(76 - S), 1 * 10^(76 - S) )

    其中：
    P代表精度，决定总位数（整数部分+小数部分），取值范围是1～76  
    S代表规模，决定小数位数，取值范围是0～P  
    根据P的范围，可以有如下的等同写法：  
   
   | P | 取值 | 原生写法示例 | 等同于 |
   | -------- | -------- | -------- | --------  | 
   |  [ 1 : 9 ] | Decimal(9,2) | Decimal32(2) |
   |  [ 10 : 18 ] | Decimal(18,2) | Decimal64(2) |
   | [ 19 : 38 ] | Decimal(38,2) | Decimal128(2) |
   | [ 39 : 76 ] | Decimal(76,2) | Decimal256(2) |
   
2. 字符串类型
   1. String（长度不限）
   2. FixedString(N)（使用null字节填充末尾字符）
   3. UUID
3. 时间类型
   1. DateTime
   2. DateTime64（记录到亚秒，2021-03-24 20:29:00.00）
   3. Date
### 复合类型
1. Array
2. Tuple
3. Enum（性能考虑，操作中会使用Int类型计算）
4. Nested 嵌套类型，只支持一级嵌套
### 特殊类型
1. Nullable 例如：Nullable(UInt8)，表示该字段可以为null，不建议使用
2. Domain 例如：IPV4 IPV6
## 库表创建
1. 数据库创建
语法如下：
   ``` sql
    CREATE DATABASE [IF NOT EXISTS] db_name
   ``` 
2. 数据表创建
   1. 常规定义方式
    ``` sql
   CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
    (
        name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
        name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
        ...
    ) ENGINE = engine
    ``` 
    mergetree
    ``` sql
    CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
    (
        name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
        name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
        ...
        INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
        INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
    ) ENGINE = MergeTree()
    ORDER BY expr
    [PARTITION BY expr]
    [PRIMARY KEY expr]
    [SAMPLE BY expr]
    [TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]
    [SETTINGS name=value, ...]
    ``` 
   2. 复制其他表结构
    ``` sql
    CREATE TABLE [IF NOT EXISTS] [db.]table_name AS [db2.]name2 [ENGINE = engine]
    ``` 
   3. 通过SELECT子句形式创建
    ``` sql
    CREATE TABLE [IF NOT EXISTS] [db.]table_name ENGINE = engine AS SELECT ... 
    ``` 
   4. 删除表
    ``` sql
    DROP TABLE [IF NOT EXISTS] [db.]table_name 
    ``` 
3. 临时表创建
   仅支持Memory表引擎，生命周期与会话绑定，会话结束，表销毁。同时临时表的优先级大于普通表。
    ``` sql
   CREATE TEMPORARY TABLE [IF NOT EXISTS] table_name
    (
        name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
        name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
        ...
    )
    ``` 
4. 分区表创建
5. 视图（普通视图和物化视图）
## 数据表基本操作
基本与MySQL一致的不再描述，增加描述常用的，但是与MySQL不同的
1. RENAME  
   重命名一个或多个表  
    ``` sql
   RENAME TABLE [db11.]name11 TO [db12.]name12, [db21.]name21 TO [db22.]name22, ... [ON CLUSTER cluster]
    ``` 
2. 查询分区
    ``` sql
    SELECT table, name, partition, active
    FROM system.parts
    WHERE database='default' and table='xxx';
    ``` 
3. 复制分区数据  
   要求两张表拥有相同的分区键并且表结构完全相同才可以复制
    ``` sql
    ALTER TABLE B REPLACE PARTITION partiton_expr FROM A
    ``` 
## 数据的删除和修改（异步执行，不可回退）
1. 删除数据
    ``` sql
    ALTER TABLE [db.]table_name DELETE WHERE filter_expr
    ``` 
2. 更新数据
    ``` sql
    ALTER TABLE [db.]table_name UPDATE column1=expr1 [, ...] WHERE filter_expr
    ``` 
## [数据字典](https://clickhouse.tech/docs/zh/sql-reference/dictionaries/)
　　ClickHouse提供的一种键值和属性映射的存储媒介，会被ClickHouse加载到内存中使用，并支持动态更新。由于常驻内存的特性，特别适合保存常量和经常使用的维度表数据，以避免不必要的JOIN查询。  
　　字典分为内部字典（ClickHouse自带）和外部字典，我们主要使用外部字典。    
``` sql
CREATE DICTIONARY dict_name
(
    ... -- attributes
)
PRIMARY KEY ... -- complex or single key configuration
SOURCE(...) -- Source configuration
LAYOUT(...) -- Memory layout configuration
LIFETIME(...) -- Lifetime of dictionary in memory
``` 
- dict_name：字典名称，全局唯一，不可重复
- [layout](https://clickhouse.tech/docs/zh/sql-reference/dictionaries/external-dictionaries/external-dicts-dict-layout/)：字典的类型，决定了数据在内存中以什么结构组织和存储
- [source](https://clickhouse.tech/docs/zh/sql-reference/dictionaries/external-dictionaries/external-dicts-dict-sources/)：字典的数据源，目前有文件、数据库等数据来源
- lifetime:字典的更新时间，例如：LIFETIME(MIN 300 MAX 360)

[使用外部字典的函数](https://clickhouse.tech/docs/zh/sql-reference/functions/ext-dict-functions/)  
其中id必须是UInt64,例如
``` sql
SELECT dictGetString('dict_name', 'attr_name', toUInt64(1))
``` 
## [表引擎](https://clickhouse.tech/docs/zh/engines/table-engines/)
表引擎（即表的类型）决定了：

- 数据的存储方式和位置，写到哪里以及从哪里读取数据
- 支持哪些查询以及如何支持。
- 并发数据访问。
- 索引的使用（如果存在）。
- 是否可以执行多线程请求。
- 数据复制参数。

　　ClickHouse有很多的表引擎，其中合并树引擎MergeTree及其家族系列最为强大，在生产环境的绝大部分场景中，都会使用MergeTree系列的表引擎。
### 引擎类型
#### MergeTree
适用于高负载任务的最通用和功能最强大的表引擎。这些引擎的共同特点是可以快速插入数据并进行后续的后台数据处理。 MergeTree系列引擎支持数据复制（使用Replicated* 的引擎版本，该版本增加了分布式协同的能力，要借助ZooKeeper的消息日志广播，实现副本实例之间的数据同步），分区和一些其他引擎不支持的其他功能。

该类型的引擎：
- [MergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/mergetree/#mergetree)
- [ReplacingMergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/replacingmergetree/#replacingmergetree)
- [SummingMergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/summingmergetree/#summingmergetree)
- [AggregatingMergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/aggregatingmergetree/#aggregatingmergetree)
- [CollapsingMergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/collapsingmergetree/#table_engine-collapsingmergetree)
- [VersionedCollapsingMergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/versionedcollapsingmergetree/#versionedcollapsingmergetree)
- [GraphiteMergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/graphitemergetree/#graphitemergetree)
  
组合关系如下：
  ![MergeTree引擎组合关系](/images/20210324/MergeTreeFamily.jpg)

我们到底应该使用哪一种表引擎?  
摘抄自 [ClickHouse各种MergeTree的关系与作用](https://mp.weixin.qq.com/s/0bXEfAN3xRXOWgGKr2GQZg)  

按照使用的场景划分，可以将上述14种表引擎大致分成以下6类应用场景:  
1. 默认情况  
  在没有特殊要求的场合，使用基础的MergeTree表引擎即可，它不仅拥有高效的性能，也提供了所有MergeTree共有的基础功能，包括列存、数据分区、分区索引、一级索引、二级索引、TTL、多路径存储等等。
  ![MergeTree](/images/20210324/MergeTree.jpg)

  与此同时，它也定义了整个MergeTree家族的基调，例如：  
  - ORDER BY 决定了每个分区中数据的排序规则;如果没有使用 PRIMARY KEY 显式的指定主键，ClickHouse 会使用排序键作为主键。
  - PRIMARY KEY 决定了一级索引(primary.idx);如果要 选择与排序键不同的主键，可选。  
  - ORDER BY 可以指代PRIMARY KEY, 通常只用声明ORDER BY 即可。  

  接下来将要介绍的其他表引擎，除开ReplicatedMergeTree系列外，都是在Merge合并动作时添加了各自独有的逻辑。  

2. 数据去重     
  通过刚才的说明，大家应该明白，MergeTree的主键（PRIMARY KEY）只是用来生成一级索引（primary.idx）的，并没有唯一性约束这样的语义。    
  一些朋友在使用MergeTree的时候，用传统数据库的思维来理解MergeTree就会出现问题。    
  如果业务上不允许数据重复，遇到这类场景就可以使用ReplacingMergeTree，如下图所示:    
  ![ReplacingMergeTree](/images/20210324/ReplacingMergeTree.jpg)
  ReplacingMergeTree通过ORDER BY，表示判断唯一约束的条件。当分区合并之时，根据ORDER BY排序后，相邻重复的数据会被排除。  

  由此，可以得出几点结论:

  第一，使用ORDER BY作为特殊判断标识，而不是PRIMARY KEY。关于这一点网上有一些误传，但是如果理解了ORDER BY与PRIMARY KEY的作用，以及合并逻辑之后，都能够推理出应该是由ORDER BY决定。

  ORDER BY的作用， 负责分区内数据排序;

  PRIMARY KEY的作用， 负责一级索引生成;

  Merge的逻辑， 分区内数据排序后，找到相邻的数据，做特殊处理。

  第二，只有在触发合并之后，才能触发特殊逻辑。以去重为例，在没有合并的时候，还是会出现重复数据。

  第三，只对同一分区内的数据有效。以去重为例，只有属于相同分区的数据才能去重，跨越不同分区的重复数据不能去重。

  上述几点结论，适用于包含ReplacingMergeTree在内的6种MergeTree，所以后面不在赘述。  

3. 预聚合(数据立方体)    
  有这么一类场景，它的查询主题是非常明确的，也就是说聚合查询的维度字段是固定，并且没有明细数据的查询需求，这类场合就可以使用SummingMergeTree或是AggregatingMergeTree，如下图所示：  
  ![AggregatingMergeTree](/images/20210324/AggregatingMergeTree.jpg)

  可以看到，在新分区合并后，在同一分区内，ORDER BY条件相同的数据会进行合并。如此一来，首先表内的数据行实现了有效的减少，其次度量值被预先聚合，进一步减少了后续计算开销。  

  聚合类MergeTree通常可以和表引擎协同使用，如下图所示：

  ![AggregatingMergeTree2](/images/20210324/AggregatingMergeTree2.jpg)

  可以将物化视图设置成聚合类MergeTree，将其作为固定主题的查询表使用。

  值得一提的是，通常只有在使用SummingMergeTree或AggregatingMergeTree的时候，才需要同时设置**ORDER BY**与**PRIMARY KEY**。

  显式的设置**PRIMARY KEY**，是为了将主键和排序键设置成不同的值，是进一步优化的体现。

  例如某个场景的查询需求如下:
  聚合条件，GROUP BY A，B，C
  过滤条件，WHERE A

  此时，如下设置将会是一种较优的选择:
  GROUP BY A，B，C
  PRIMARY KEY A

  BTW，如果**ORDER BY**与**PRIMARY KEY**不同，**PRIMARY KEY**必须是**ORDER BY**的前缀(为了保证分区内数据和主键的有序性)。

4. 数据更新  
数据的更新在ClickHouse中有多种实现手段，例如按照分区Partition重新写入、使用Mutation的DELETE和UPDATE查询。

使用CollapsingMergeTree或VersionedCollapsingMergeTree也能实现数据更新，这是一种使用标记位，以增代删的数据更新方法，如下图所示：

![CollapsingMergeTree](/images/20210324/CollapsingMergeTree.jpg)

通过增加一个标志字段(例如图中的sigh字段)，作为数据有效性的判断依据。

可以看到，在新分区合并后，在同一分区内，ORDER BY条件相同的数据，其标志值为1和-1的数据行会进行抵消。

下图是另外一种便于理解的视角，就如同挤压瓦楞纸一般，数据被抵消了:
![CollapsingMergeTree2](/images/20210324/CollapsingMergeTree2.jpg)

**CollapsingMergeTree和VersionedCollapsingMergeTree的区别又是什么呢?**

**CollapsingMergeTree**对数据**写入的顺序是敏感**的，它要求标志位需要按照正确的顺序排序。例如按照1，-1的写入顺序是正确的; 而如果按照-1，1的错误顺序写入，CollapsingMergeTree就无法正确抵消。

试想，如果在一个多线程并行的写入场景，我们是无法保证这种顺序写入的，此时就需要使用VersionedCollapsingMergeTree了。

**VersionedCollapsingMergeTree**在CollapsingMergeTree基础之上，额外要求指定一个version字段，在分区Merge合并时，它会自动将version字段追加到ORERY BY的末尾，从而保证了标志位的有序性。
``` sql
ENGINE = VersionedCollapsingMergeTree(sign,ver)
ORDER BY id
//等效于
ORDER BY id,ver
```

5. 监控集成  
GraphiteMergeTree可以与Graphite集成，如果你使用了Graphite作为系统的运行监控系统, 则可以通过GraphiteMergeTree存储指标数据，加速查询性能、降低存储成本。

6. 高可用  
Replicated* 拥有数据副本的能力，如下图所示:
![ReplicatedMergeTree](/images/20210324/ReplicatedMergeTree.jpg)

结合刚才的5类场景，如果进一步需要高可用的需求，选择一种MergeTree和Replicated组合即可，例如ReplicatedMergeTree、ReplicatedReplacingMergeTree等等。  

#### 日志 
具有最小功能的轻量级引擎。当您需要快速写入许多小表（最多约100万行）并在以后整体读取它们时，该类型的引擎是最有效的。

该类型的引擎：
- TinyLog
- StripeLog
- Log
  
#### 集成引擎 
用于与其他的数据存储与处理系统集成的引擎。
该类型的引擎：
- Kafka
- MySQL
- ODBC
- JDBC
- HDFS
  
#### 用于其他特定功能的引擎 
该类型的引擎：
- Distributed
- MaterializedView
- Dictionary
- Merge
- File
- Null
- Set
- Join
- URL
- View
- Memory
- Buffer