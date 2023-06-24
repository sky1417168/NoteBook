# Hudi基本概念

## 发展历史

2015 年：发表了增量处理的核心思想/原则（O'reilly 文章）。

2016 年：由 Uber 创建并为所有数据库/关键业务提供支持。

2017 年：由 Uber 开源，并支撑 100PB 数据湖。

2018 年：吸引大量使用者，并因云计算普及。

2019 年：成为 ASF 孵化项目，并增加更多平台组件。

2020 年：毕业成为 Apache 顶级项目，社区、下载量、采用率增长超过 10 倍。

2021 年：支持 Uber 500PB 数据湖，SQL DML、Flink 集成、索引、元服务器、缓存。

## Hudi特性

1）可插拔索引机制支持快速Upsert/Delete。

2）支持增量拉取表变更以进行处理。

3）支持事务提交及回滚，并发控制。

4）支持Spark、Presto、Trino、Hive、Flink等引擎的SQL读写。

5）自动管理小文件，数据聚簇，压缩，清理。

6）流式摄入，内置CDC源和工具。

7）内置可扩展存储访问的元数据跟踪。

8）向后兼容的方式实现表结构变更的支持

## 使用场景

1）近实时写入

- 减少碎片化工具的使用。

- CDC 增量导入 RDBMS 数据。
- 限制小文件的大小和数量。

2）近实时分析

-  相对于秒级存储（Druid, OpenTSDB），节省资源。

-  提供分钟级别时效性，支撑更高效的查询。

-  Hudi作为lib，非常轻量。

3）增量 pipeline

-  区分arrivetime和event time处理延迟数据。

-  更短的调度interval减少端到端延迟（小时 -> 分钟） => Incremental Processing。

4）增量导出

-  替代部分Kafka的场景，数据导出到在线服务存储 e.g. ES。

## 核心概念

### 基本概念

#### 时间轴-TimeLine

![时间轴-1](../images/hudi/hudi-consept-1/hudi-consept-timeline-1.png)

> [!NOTE] Hudi的核心是维护表上在不同的即时时间（instants）执行的所有操作的时间轴（timeline），这有助于提供表的即时视图，同时还有效地支持按到达顺序检索数据。

一个instant由以下三个部分组成：

1）Instant action：在表上执行的操作类型

**COMMITS**：一次commit表示将一批数据原子性地写入一个表。

**CLEANS**：清除表中不再需要的旧版本文件的后台活动。

**DELTA_COMMIT**：增量提交指的是将一批数据原子性地写入一个MOR类型的表，其中部分或所有数据可以写入增量日志。

**COMPACTION**：合并Hudi内部差异数据结构的后台活动，例如:将更新操作从基于行的log日志文件合并到列式存储的数据文件。在内部，COMPACTION体现为timeline上的特殊提交。

**ROLLBACK**：表示当commit/delta_commit不成功时进行回滚，其会删除在写入过程中产生的部分文件。

**SAVEPOINT**：将某些文件组标记为已保存，以便其不会被删除。在发生灾难需要恢复数据的情况下，它有助于将数据集还原到时间轴上的某个点。

2）Instant time

通常是一个时间戳（例如：20190117010349），它按照动作开始时间的顺序单调增加。

3）State
**REQUESTED**：表示某个action已经调度，但尚未执行。
**INFLIGHT**：表示action当前正在执行。
**COMPLETED**：表示timeline上的action已经完成。

4）两个时间概念

区分两个重要的时间概念：

**Arrival time**: 数据到达 Hudi 的时间，commit time。

**Event time**: record 中记录的时间。

![时间轴-2](../images/hudi/hudi-consept-1/hudi-consept-timeline-2.png ':size=70%')

> [!NOTE] 上图中采用时间（小时）作为分区字段，分别有7:00，8:00，9:00，10:00等多个分区
>
> 从 10:00 开始陆续产生各种 commits
>
> 如果10:20 来了3条7:00的数据，2条8:00的数据，1条 9:00 的数据，根据event time该数据仍然可以落到对应的分区
>
> 通过 timeline 直接消费 10:00 （commit time）之后的增量更新，那么这些延迟的数据仍然可以被消费到。

#### 文件布局-File Layout

Hudi将一个表映射为如下文件结构：

![文件布局](../images/hudi/hudi-consept-1/fhudi-consept-filelayout-3.png ':size=30%')

**Hudi存储分为两个部分**：

（1）元数据：.hoodie目录对应着表的元数据信息，包括表的版本管理（Timeline）、归档目录（存放过时的instant也就是版本），一个instant记录了一次提交（commit）的行为、时间戳和状态，Hudi以时间轴的形式维护了在数据集上执行的所有操作的元数据

（2）数据：和hive一样，以分区方式存放数据，分区里面存放着Base File（.parquet）和Log File（.log.*）

**Hudi的文件管理**：

（1）Hudi将数据表组织成分布式文件系统基本路径（basepath）下的目录结构

（2）表被划分为多个分区，这些分区是包含该分区的数据文件的文件夹，非常类似于Hive表

（3）在每个分区中，文件被组织成文件组，由文件ID唯一标识

（4）每个文件组包含几个文件片（FileSlice）

（5）每个文件片包含：

-  一个基本文件(.parquet)：在某个commit/compaction即时时间（instant time）生成的（MOR可能没有）

-  多个日志文件(.log.*)，这些日志文件包含自生成基本文件以来对基本文件的插入/更新（COW没有）

> [!TIP] base文件就是parquet文件
>
> base文件的命名：\<fileId>\_\<writeToken>\_\<instant>.\<fileExtension>
>
> log文件的命名：.\<fileId>\_\<instant>.\<fileExtension>.\<version>_\<writeToken>
>
> writeToken：\<taskPartitionId>\-\<stageId>\-\<taskAttemptId>

文件组的布局：

![文件组](../images/hudi/hudi-consept-1/hudi-consept-fliegroup-4.png ':size=50%')

> [!NOTE] 如上图，一个文件组（FileGroup）下有多个文件片（FileSlice），一个文件片包含多个base文件（parquet）和若干个log文件（MOR表才有）

（6）Hudi采用了多版本并发控制(Multiversion Concurrency Control, MVCC)

-  compaction操作：合并日志和基本文件以产生新的文件片
-  clean操作：清除不使用的/旧的文件片以回收文件系统上的空间

![数据更新](../images/hudi/hudi-consept-1/hudi-consept-update-5.png ':size=60%')

> [!NOTE]
>
> -  Upsert操作开始，数据被划分为Insert和Updates两类数据
>
> - 10:10Commit提交
> - 文件组有fileid1,fileid2,fileid3,fileid4  每个组下分Base Files和Log File
>
> - 标红的是10:00更新的，只在fileid2和fileid4下有更新
>
> - 表绿的是10:05分更新的，在时间轴上的也是10:05
>
> - 右侧分别是两种读取方式：
>
>   - 读优化查询，只读取了parquet文件
>   - 近事实查询，合并了log文件

（7）Hudi的base file（parquet 文件）在 footer 的 meta 去记录了 record key 组成的 BloomFilter，用于在 file based index 的实现中实现高效率的 key contains 检测。只有不在 BloomFilter 的 key 才需要扫描整个文件消灭假阳。

（8）Hudi 的 log （avro 文件）是自己编码的，通过积攒数据 buffer 以 LogBlock 为单位写出，每个 LogBlock 包含 magic number、size、content、footer 等信息，用于数据读、校验和过滤。

![文件格式](../images/hudi/hudi-consept-1/hudi-consept-file-6.png ':size=40%')

> [!NOTE] 
>
> 分别是hudi组织下的parquet文件和log文件格式

#### 索引-Index

1）原理
Hudi通过索引机制提供高效的upserts，具体是将给定的hoodie key(record key + partition path)与文件id（文件组）建立唯一映射。这种映射关系，数据第一次写入文件后保持不变，所以，一个 FileGroup 包含了一批 record 的所有版本记录。Index 用于区分消息是 INSERT 还是 UPDATE。

![Hudi索引](../images/hudi/hudi-consept-1/hudi-consept-index-7.png ':size=50%')

Hudi 为了消除不必要的读写，引入了索引的实现。在有了索引之后，更新的数据可以快速被定位到对应的 File Group。上图为例，白色是基本文件，黄色是更新数据，有了索引机制，可以做到：避免读取不需要的文件、避免更新不必要的文件、无需将更新数据与历史数据做分布式关联，只需要在 File Group 内做合并。



2）索引选项

| **Index****类型**        | **原理**                                                     | **优点**                                                  | **缺点**                               |
| ------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- | -------------------------------------- |
| Bloom  Index             | 默认配置，使用布隆过滤器来判断记录存在与否，也可选使用record key的范围裁剪需要的文件 | 效率高，不依赖外部系统，数据和索引保持一致性              | 因假阳性问题，还需回溯原文件再查找一遍 |
| Simple  Index            | 把update/delete操作的新数据和老数据进行join                  | 实现最简单，无需额外的资源                                | 性能比较差                             |
| HBase  Index             | 把index存放在HBase里面。在插入 File Group定位阶段所有task向HBase发送 Batch Get 请求，获取 Record Key 的 Mapping 信息 | 对于小批次的keys，查询效率高                              | 需要外部的系统，增加了运维压力         |
| Flink  State-based Index | HUDI  在 0.8.0 版本中实现的 Flink witer，采用了 Flink 的 state 作为底层的 index 存储，每个 records 在写入之前都会先计算目标 bucket ID。 | 不同于 BloomFilter Index，避免了每次重复的文件 index 查找 |                                        |

​    注意：Flink只有一种state based index（和bucket_index），其他index是Spark可选配置。

3）全局索引与非全局索引

全局索引：全局索引在全表的所有分区范围下强制要求键的唯一性，也就是确保对给定的键有且只有一个对应的记录。全局索引提供了更强的保证，但是随着表增大，update/delete 操作损失的性能越高，因此更适用于小表。

非全局索引：默认的索引实现，只能保证数据在分区的唯一性。非全局索引依靠写入器为同一个记录的update/delete提供一致的分区路径，同时大幅提高了效率，更适用于大表。

从index的维护成本和写入性能的角度考虑，维护一个global index的难度更大，对写入性能的影响也更大，所以需要non-global index。

HBase索引本质上是一个全局索引，bloom和simple index都有全局选项：

 hoodie.index.type=GLOBAL_BLOOM

 hoodie.index.type=GLOBAL_SIMPLE

4）索引的选择策略

（1）对事实表的延迟更新

许多公司会在NoSQL数据存储中存放大量的交易数据。例如共享出行的行程表、股票买卖记录的表、和电商的订单表。这些表通常一直在增长，且大部分的更新随机发生在较新的记录上，而对旧记录有着长尾分布型的更新。这通常是源于交易关闭或者数据更正的延迟性。换句话说，大部分更新会发生在最新的几个分区上而小部分会在旧的分区。

对于这样的作业模式，布隆索引就能表现地很好，因为查询索引可以靠设置得当的布隆过滤器来裁剪很多数据文件。另外，如果生成的键可以以某种顺序排列，参与比较的文件数会进一步通过范围裁剪而减少。Hudi用所有文件的键域来构造区间树，这样能来高效地依据输入的更删记录的键域来排除不匹配的文件。

为了高效地把记录键和布隆过滤器进行比对，即尽量减少过滤器的读取和均衡执行器间的工作量，Hudi缓存了输入记录并使用了自定义分区器和统计规律来解决数据的偏斜。有时，如果布隆过滤器的假阳性率过高，查询会增加数据的打乱操作。Hudi支持动态布隆过滤器（设置hoodie.bloom.index.filter.type=DYNAMIC_V0）。它可以根据文件里存放的记录数量来调整大小从而达到设定的假阳性率。

![数据更新](../images/hudi/hudi-consept-1/hudi-consept-index-8.gif)


（2）对事件表的去重

事件流无处不在。从Apache Kafka或其他类似的消息总线发出的事件数通常是事实表大小的10-100倍。事件通常把时间（到达时间、处理时间）作为首类处理对象，比如物联网的事件流、点击流数据、广告曝光数等等。由于这些大部分都是仅追加的数据，插入和更新只存在于最新的几个分区中。由于重复事件可能发生在整个数据管道的任一节点，在存放到数据湖前去重是一个常见的需求。

总的来说，低消耗去重是一个非常有挑战的工作。虽然可以用一个键值存储来实现去重（即HBase索引），但索引存储的消耗会随着事件数增长而线性增长以至于变得不可行。事实上，有范围裁剪功能的布隆索引是最佳的解决方案。我们可以利用作为首类处理对象的时间来构造由事件时间戳和事件id（event_ts+event_id)组成的键，这样插入的记录就有了单调增长的键。这会在最新的几个分区里大幅提高裁剪文件的效益。


![数据更新-2](../images/hudi/hudi-consept-1/hudi-consept-index-9.gif)

（3）对维度表的随机更删

正如之前提到的，如果范围比较不能裁剪许多文件的话，那么布隆索引并不能带来很好的效益。在这样一个随机写入的作业场景下，更新操作通常会触及表里大多数文件从而导致布隆过滤器依据输入的更新对所有文件标明阳性。最终会导致，即使采用了范围比较，也还是检查了所有文件。使用简单索引对此场景更合适，因为它不采用提前的裁剪操作，而是直接和所有文件的所需字段连接。如果额外的运维成本可以接受的话，也可以采用HBase索引，其对这些表能提供更加优越的查询效率。

当使用全局索引时，也可以考虑通过设置hoodie.bloom.index.update.partition.path=true或hoodie.simple.index.update.partition.path=true来处理 的情况；例如对于以所在城市分区的用户表，会有用户迁至另一座城市的情况。这些表也非常适合采用Merge-On-Read表型。

![数据更新-3](../images/hudi/hudi-consept-1/hudi-consept-index-10.gif)

#### 表类型-Table Type



#### 查询类型-Query Types



### 数据写



### 数据读



!> 提示信息

