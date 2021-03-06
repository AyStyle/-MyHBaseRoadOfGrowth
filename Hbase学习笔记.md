# HBase学习笔记

## 第一部分 初识HBase
### 第一节 HBase简介
#### 1.1 HBase简介 
```
    HBase是基于Google的BigTable论文而来的，是一个分布式海量列式非关系型分布式数据库系统，
可以提供超大规模数据集的实时随机读写
```

#### 1.2 HBase的特点
+ 海量存储：底层基于HDFS存储海量数据
+ 列式存储：HBase表的数据是基于列族进行存储的，一个列族包含若干列
+ 极易扩展：底层依赖HDFS，当磁盘空间不足的时候，只需要动态增加DataNode服务节点即可 
+ 高并发：支持高并发的读写请求
+ 稀疏：稀疏主要是针对HBase列的灵活性，在列族中可以指定任意多的列，在列数据为空的情况下，是不会占用存储空间的
+ 数据的多版本：HBase表中的数据可以有多个版本值，默认情况下是根据版本号去区分，版本号就是插入数据的时间戳
+ 数据类型单一：所有的数据在HBase中是以字节数组进行存储的

#### 1.3 HBase的应用场景
Hbase适合海量明细数据的存储，并且后期需要有很好的查询性能

+ 交通方面：船舶GPS信息，每天有上千万左右的数据存储
+ 金融方面：消费信息、贷款信息、信用卡还款信息等
+ 电商方面：电商网站的交易信息、物流信息、游览信息等
+ 电信方面：通话信息

### 第二节 HBase数据模型
|概念|描述|
|:---:|:---|
|NameSpace|命名空间，类似于关系型数据库的database概念，每个命名空间下有多个表|
|Table|类似于关系型数据库中的表。不同的是，HBase定义表时只需要声明列族即可，数据属性，例如：超时时间（TTL）、压缩算法（COMPRESSION）等都在列族中定义，不需要具体声明列|
|Row|Hbase中表的每行数据都有一个RowKey和多个Column组成。一个行包含了多个列|
|RowKey|RowKey由用户指定的一串不重复的字符串定义，是一行的唯一标识！数据是按照RowKey的字典顺序存储的，并且查询数据时只能根据RowKey进行检索，所以RowKey的设计十分重要|
|ColumnFamily|列族是多个列的集合。一个列族可以动态地灵活定义多个列。表的相关属性大部分都定义在列族上，同一个表里的不同列族可以有完全不同的属性配置，但是同一个列族内所有列都会有相同的属性。列族存在的意义就是HBase会把相同列族的列尽量放在同一台机器上|
|Column Qualifier|Hbase中的列是可以随意定义的，一个行中的列不限名字、不限数量，只限定列族。因此列必须依赖于列族存在！列的名称前必须带有其列族|
|TimeStamp|用于表示数据的不同版本。时间戳默认由系统指定，也可以由用户显式指定|
|Cell|一个列中可以存储多个版本的数据。每个版本就称为一个单元格|
|Region|Region由一个表的若干行组成！在Region中行的排序按照行键（RowKey）字典排序。Region不能跨RegionServer，且数据量大的时候，HBase会拆分Region|

#### 第三节 HBase整体架构
+ Zookeeper
   + 实现了HMaster的高可用
      + 保存了HBase的元数据信息，是所有HBase表的寻址入口
   + 对HMaster和HRegionServer实现了监控
    
+ HMaster
   + 为HRegionServer分配Region
      + 维护整个集群的负载均衡
   + 维护集群的元数据信息
   + 发现失效的Region，并将失效的Region分配到正常的HRegionServer上
    
+ HRegionServer
   + 负责管理Region
   + 接受客户端的读写数据请求
   + 切分运行过程中变大的Region
    
+ Region
   + 每个HRegion由多个Store构成
   + 每个Store保存一个列族（Columns Family），表有几个列族，则有几个Store
   + 每个Store由一个MemStore和多个StoreFile组成，MemStore是Store在内存中的内容，写到文件后就是StoreFile。StoreFile底层是以HFile的格式保存 
    
## 第二部分 HBase原理深入
### 第一节 HBase读数据流程
1. 首先从ZK找到Hbase表对应Meta表的RegionServer位置
2. 请求Meta表的RegionServer服务器，并根据要查询的namespace、表名和RowKey信息，找到写入数据对应的Region信息
3. 向Region所在的RegionServer服务器发起请求，并从Region上获取数据
    1. 先查询MemStore，如果有数据就直接返回
    2. 如果MemStore没有，那么查询BlockCache，如果有就直接返回
    3. 如果BlockCache没有，那么查询StoreFile，并把查询结果在BlockCache上缓存，然后返回给客户端
    
### 第二节 HBase写数据流程
1. 首先从ZK找到Hbase表对应Meta表的RegionServer位置
2. 请求Meta表的RegionServer服务器，并根据要查询的namespace、表名和RowKey信息，找到写入数据对应的Region信息
3. 向Region所在的RegionServer服务器发起请求，并将数据写入到Region上
  1. 先写HLog/WAL（Write ahead log），再写MemStore
  2. 当MemStore达到阈值后，那么就把数据刷到磁盘上生成StoreFile
  3. 生成StoreFile后就删除HLog中的历史数据

### 第三节 Hbase的flush（刷写）及compact（合并）机制
#### 刷写机制
1. 当memstore的大小超过默认的128M，会刷写到磁盘
2. 当memstore中的数据时间超过默认的1小时，会刷写到磁盘
3. HRegionServer的全局memstore大小，超过堆的默认40%时，会刷写到磁盘
4. 手动flush

#### 阻塞机制
1. memstore中的数据达到512MB
2. RegionServer全部memstore达到阈值时，默认堆内存的0.95
```
触发条件：
   定期检查线程，间隔10秒一次
```

#### 合并机制
+ minor compact：
    1. 待合并文件必须大于等于默认值3，小于等于默认值10
    2. 文件小于默认128MB时，会被选中合并
    3. 文件大于阈值时，不会被选中合并，默认Long.MAX_VALUE
    ```
    触发条件：
       1. memstore刷写磁盘时
       2. 定期检查线程，间隔10秒一次
    ```

+ major compact：
  ```
  合并Store中的所有HFile为一个HFile。
  
  这个过程会真正删除有标记为删除的数据，同时超过单元格maxVersion的版本数据也会被删除。
  合并的频率比较低，默认7天一次，并且性能消耗非常大，生成环境通常关闭（设置为0）
  
  手动触发：major_compact tableName
  ```

### 第四节 Region拆分机制
#### 4.1 拆分策略
1. ConstantSizeRegionSplitPolicy
   ```
   0.94版本之前的默认切分策略
   当Region大于阈值（10G）之后就会触发切分，一个region切分成2个
   
   这种策略在生产环境有弊端：切分策略对于大表和小表没有明显的区分
      1. 阈值较大对大表友好，但是小表就可能不会触发分裂，极端情况就是1个
      2. 阈值较小对小表友好，但是大表会在整个集群产生大量的Region，这对于集群的管理、资源使用、failover来说都不是好事
   ```
   
2. IncreasingToUpperBoundRegionSplitPolicy
   ```
   0.94版本-2.0版本的默认切分策略
   
   公式：切分大小 = RegionServer上同一个表的region个数的三次方 + 128MB * 2，当这个值大于10G时，取10G
   ```

3. SteppingSplitPolicy
   ```
   2.0版本的默认切分策略
   
   当前RegionServer上同一个表的region个数为1时，按照flush size * 2切分，否则按照MaxRegionFileSize切分
   ```

4. KeyPrefixRegionSplitPolicy
   ```
   根据RowKey的前缀对数据进行分组，这里指定RowKey的前多少位作为前缀
   ```

5. DelimitedKeyPrefixRegionSplitPolicy
   ```
   保证相同前缀的数据在同一个Region中，这里指定RowKey的切分规则
   ```

6. DisabledRegionSplitPolicy
   ```
   不启动自动拆分，需要指定手动拆分
   ```

#### 4.2 RegionSplitPolicy的应用
Region拆分策略可以全局统一配置，也可以为单独的表指定拆分策略

1. 通过hbase-site.xml全局统一配置（对hbase所有表生效）
2. 通过Java API为单独的表指定Region拆分策略
3. 通过Hbase Shell为单个表指定Region拆分策略

### 第五节 Hbase表的预分区
#### 5.1 为什么要预分区？
```
    当一个table刚被创建的时候，Hbase默认的分配一个Region给table。
也就是说这个时候，所有的读写请求都会访问到同一个RegionServer的同一个Region上。
这个时候就达不到负载均衡的效果了，集群中的其他RegionServer就可能处于比较空闲的状态
```

#### 5.2 预分区的优点
1. 增加数据读写效率
2. 负载均衡，防止数据倾斜
3. 方便集群容灾调度Region
   ```
   每个Region维护着StartRow与EndRow，如果加入的数据符合某个Region维护的RowKey范围，
   则该数据交给这个region维护
   ```
   
### 第六节 Region合并
#### 6.1 Region合并说明
```
Region的合并并不是为了性能，而是处于维护的目的
```

#### 6.2 如何进行Region合并
1. 通过Merge类冷合并Region
   ```
   需要先关闭hbase集群
   ```
2. 通过online_merge热合并Region
   ```
   不需要先关闭hbase集群
   ```

## 第七部分 API操作
### 第二节 Hbase协处理器
#### 2.1 协处理器概述
```
   访问HBase的方式是使用Scan或get获取数据，在获取到的数据上进行业务运算。
但是在数据量非常大的时候，比如一个有上亿行及十万个列的数据集，再按照常用的方式获取数据就会遇到性能问题。
客户端也需要有强大的计算能力以及足够的内存来处理这么多的数据。

   使用Coprocessor（协处理器）。将业务运算代码封装到Coprocessor中并在RegionServer上运行，即在数据
实际存储位置执行，最后将运算结果返回到客户端。利用协处理器，用户可以编写运行在HbaseServer端的代码。

Hbase Coprocessor类似一下概念：
   1. 触发器和存储过程
   2. MapReduce
   3. AOP
```

#### 2.2 协处理器类型
+ Observer Coprocessor
  ```
  Observer Coprocessor与触发器（Trigger）类似：在一些特定事件发生时回调函数（也成为了Hook）被执行。
  这些事件包括一些用户产生的事件，也包括服务器端内部自动产生的事件。
  
  常见Observer Coprocessor协处理器类型：
     RegionObserver：用户可以用这种处理器处理数据修改事件，他们与表的Region联系紧密
     MasterObserver：可以被用作管理或DDL类型的操作，这些是集群级事件
     WALObserver：提供控制WAL的钩子函数
  ```
  
+ Endpoint Coprocessor
  ```
  这类协处理器类似传统数据库中的存储过程，客户端可以调用这些Endpoint协处理器在
  RegionServer中执行一段代码，并将结果返回给客户端进一步处理。
  
  ```

### 第四节 Hbase表RowKey设计
1. RowKey长度原则：
   ```
   RowKey是一个二进制码流，可以是任意字符串，最大长度64kb，实际应用中一般为10-100bytes，
   以byte[]形式保存，一般设计成定长。
   
   建议越短越好，不要超过16个字节。过长会降低MemStore内存的利用率和HFile存储数据的效率
   ```

2. RowKey散列原则
   ```
   建议将RowKey的高位作为散列字段，这样将提高数据均衡分布在每个Regionserver，以实现负载均衡
   ```

3. RowKey唯一原则
   ```
   必须在设计上保证其唯一性
   
   访问Hbase表中的行，有三种方式：
     1. 单个RowKey
     2. RowKey的range
     3. 全表扫描（一定要避免全表扫描）
   ```
   
4. RowKey排序原则
   ```
   RowKey按照ASCII码字典顺序排序，经常一起查询的数据建议放到一起
   
   先比较第一个字节，如果第一个字节相同，则比较第二个字节，一次类推
   如果到第X个字节的时候，其中一个已经超过RowKey的长度，那么短的排在前面
   ```

### 第五节 Hbase表的热点
#### 5.1 什么是热点
```
   检索Hbase的记录首先要通过RowKey来定位数据行。当大量的client访问Hbase集群的一个或少数几个节点时，
造成少数RegionServer的读/写请求过多，负载过大，而其他RegionServer负载缺很小，就造成了热点现象
```

#### 5.2 热点的解决方案
+ 预分区
   ```
   预分区的目的让表的数据可以均衡的分散在集群中，而不是默认只有一个Region分布在集群的一个节点上
   ```
  
+ 加盐
   ```
   在RowKey前面增加随机数，具体就是给RowKey分配一个随机前缀以使得它和之前的RowKey开头不同
   ```

+ 哈希
   ```
   哈希会使得同一行永远用一个前缀加盐。哈希也可以使负载分散到整个集群，但是读却是可以预测的。
   使得确定的哈希可以让客户端重构完整的RowKey，可以使用get操作准确获取某一行数据
   ```
+ 反转
   ```
   反转固定长度或者数字格式的RowKey。这样可以使得RowKey中经常改变的部分（最没有意义的部分）
   放在前面。这样可以有效的随机RowKey，但是牺牲了RowKey的有序性
   ```

### 第六节 Hbase的二级索引
```
Hbase表按照RowKey查询性能是最高的。RowKey就相当于Hbase表的一级索引！
Hbase的二级索引其本质就是建立Hbase表中列与行键之间的映射关系。

常见的Hbase二级索引实现有：Solr、ES、Phoenix等
```

### 第七节 布隆过滤器在Hbase的应用
```

```




