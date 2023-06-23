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

Hudi的核心是维护表上在不同的即时时间（instants）执行的所有操作的时间轴（timeline），这有助于提供表的即时视图，同时还有效地支持按到达顺序检索数据。一个instant由以下三个部分组成

![时间轴](https://southwinds.oss-cn-beijing.aliyuncs.com/NoteBook/hudi/hudi-consept-1/hudi-timeline.png)

#### 文件布局-File Layout

#### 索引-Index

#### 表类型-Table Type

#### 查询类型-Query Types

### 数据写

### 数据读
