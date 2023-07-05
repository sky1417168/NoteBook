# Hudi集成Spark

### 1、前提

想要使用spark支持hudi，首先要编译出**hudi-spark3-bundle**包，然后使用spark的时候要加载这个包

例如，加载包到spark的jar目录下，这样就不需要每次启动都加载包了

```bash
hudi-spark3.3-bundle_2.12-0.12.0.jar
```

### 2、启动

详细参考官网：https://hudi.apache.org/docs/quick-start-guide

#### 启动spark-shell

```bash
# Spark 3.2
spark-shell \
  --packages org.apache.hudi:hudi-spark3.2-bundle_2.12:0.13.1 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
  --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
  --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension'
  --master yarn 
  --queue comom_offline 
```

#### 启动spark-sql

```bash
# Spark 3.2
spark-sql --packages org.apache.hudi:hudi-spark3.2-bundle_2.12:0.13.1 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
  --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
  --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog'
  --master yarn 
  --queue comom_offline 
  --num-executors 60 
  --executor-cores 2 
  --driver-memory 2G 
  --executor-memory 6G
```

> [!note]
>
> 可以在启动的时候指定资源大小，这样查询使用的资源才够用，具体大小具体设置
>
> 但是这个资源是固定的，不会释放

### 3、使用代码模式

#### pom依赖

```xml
<properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <scala.version>2.12.15</scala.version>
    <spark.version>3.3.1</spark.version>
    <hadoop.version>3.3.4</hadoop.version>
    <scala.compat.version>2.12</scala.compat.version>
    <encoding>UTF-8</encoding>
</properties>

<dependencies>
    <dependency>
        <groupId>org.scala-lang</groupId>
        <artifactId>scala-library</artifactId>
        <version>${scala.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_${scala.compat.version}</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_${scala.compat.version}</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-client</artifactId>
        <version>${hadoop.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-hive_${scala.compat.version}</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <!--工具类-->
    <dependency>
        <groupId>cn.hutool</groupId>
        <artifactId>hutool-all</artifactId>
        <version>4.5.15</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-avro_${scala.compat.version}</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <!--spark3.3需要这个包-->
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.8.1</version>
    </dependency>
</dependencies>
```

#### 插入数据案例

```scala
import org.apache.hudi.DataSourceWriteOptions
import org.apache.hudi.config.HoodieWriteConfig
import org.apache.spark.sql.{SaveMode, SparkSession}
import scala.language.postfixOps

object HudiDemo {
  def main(args: Array[String]): Unit = {
    // 1. 构建Spark运行环境 
    val spark = SparkSession
      .builder()
      .master("local[*]")
      .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .config("spark.default.parallelism", 2)
      .appName("hudi_test")
      .getOrCreate()
    
    // 本地测试
    spark.sparkContext.hadoopConfiguration.set("fs.defaultFS","hdfs://sys-test-01:9000")

    import spark.implicits._
      
    // 2.获取数据
    val orderDF = Seq(
      (1, "Tom",1001),
      (2, "Jerry",1001),
      (3, "Lucy",1001)
      ).toDF("id", "name","dt")

    // 3. 写入到Hudi
    orderDF.write.format("org.apache.hudi")
      .option(HoodieWriteConfig.INSERT_PARALLELISM_VALUE.key(), "2")
      .option(HoodieWriteConfig.UPSERT_PARALLELISM_VALUE.key(), "2")
      .option(HoodieWriteConfig.TBL_NAME.key(), "tbl_order")
      .option(DataSourceWriteOptions.TABLE_TYPE.key(), "COPY_ON_WRITE")
      .option(DataSourceWriteOptions.RECORDKEY_FIELD.key(), "id")
      .option(DataSourceWriteOptions.PRECOMBINE_FIELD.key(), "name")
      .option(DataSourceWriteOptions.PARTITIONPATH_FIELD.key(), "dt")
      .option(DataSourceWriteOptions.HIVE_STYLE_PARTITIONING.key(), value = true)
      .mode(SaveMode.Append)
      .save("/hudi/tbl_order/")
  }
}
```

> [!note]
>
> insert overwrite_table对有分区的表使用INSERT_OVERWRITE类型的写操作，而对无分区的表使用INSERT_OVERWRITE_TABLE

#### 读取数据案例

- 基本读取

```scala
import org.apache.hudi.DataSourceReadOptions
import org.apache.spark.sql.SparkSession
import scala.language.postfixOps

object HudiReadDemo {
  def main(args: Array[String]): Unit = {

    // 1. 构建Spark运行环境
    val spark = SparkSession
      .builder()
      .master("local[*]")
      .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .config("spark.default.parallelism", 2)
      .appName("hudi_test")
      .getOrCreate()

    // 本地测试
    spark.sparkContext.hadoopConfiguration.set("fs.defaultFS", "hdfs://sys-test-01:9000")

    // 读取hudi
    val df = spark.read.format("org.apache.hudi")
      .option(DataSourceReadOptions.READ_PATHS.key(), "/hudi/tbl_order/")
      .option(DataSourceReadOptions.QUERY_TYPE.key(), "snapshot")
      .option(DataSourceReadOptions.READ_PRE_COMBINE_FIELD.key(), "name")
      .load()

    df.show()
  }
}
```

- 增量读取

```scala
import org.apache.hudi.DataSourceReadOptions.{BEGIN_INSTANTTIME, QUERY_TYPE, QUERY_TYPE_INCREMENTAL_OPT_VAL}
import org.apache.spark.sql.SparkSession

import scala.language.postfixOps

object HudiIncrementalReadDemo {
  def main(args: Array[String]): Unit = {

    // 1. 构建Spark运行环境
    val spark = SparkSession
      .builder()
      .master("local[*]")
      .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .config("spark.default.parallelism", 2)
      .appName("hudi_test")
      .getOrCreate()

    // 本地测试
    spark.sparkContext.hadoopConfiguration.set("fs.defaultFS", "hdfs://sys-test-01:9000")

    import spark.implicits._
    
    // spark-shell
    // reload data
    spark.
      read.
      format("hudi").
      load("/hudi/tbl_order/").
      createOrReplaceTempView("hudi_trips_snapshot")

    val commits = spark.sql("select distinct(_hoodie_commit_time) as commitTime from  hudi_trips_snapshot order by commitTime").map(k => k.getString(0)).take(50)
    val beginTime = commits(commits.length - 2) // commit time we are interested in

    // incrementally query data
    val tripsIncrementalDF = spark.read.format("hudi").
      option(QUERY_TYPE.key(), QUERY_TYPE_INCREMENTAL_OPT_VAL).
      option(BEGIN_INSTANTTIME.key(), beginTime).
      load("/hudi/tbl_order/")
    tripsIncrementalDF.createOrReplaceTempView("hudi_trips_incremental")

    spark.sql("select `_hoodie_commit_time`, fare, begin_lon, begin_lat, ts from  hudi_trips_incremental where fare > 20.0").show()

  }
}
```

#### spark属性

主要涉及到两个类，一个是**DataSourceReadOptions**，另一个是**DataSourceWriteOptions**

- **spark读属性**：DataSourceReadOptions

  | 属性                      | key                                        | default_value | 说明                              |
  | ------------------------- | ------------------------------------------ | ------------- | --------------------------------- |
  | TIME_TRAVEL_AS_OF_INSTANT | as.of.instant                              |               | 时间旅行                          |
  | BEGIN_INSTANTTIME         | hoodie.datasource.read.begin.instanttime   |               | 读取指定时间后的数据              |
  | END_INSTANTTIME           | hoodie.datasource.read.end.instanttime     |               | 读取指定时间前的数据              |
  | READ_PATHS                | hoodie.datasource.read.paths               |               |                                   |
  | INCREMENTAL_FORMAT        | hoodie.datasource.query.incremental.format | latest_state  | latest_state、cdc                 |
  | QUERY_TYPE                | hoodie.datasource.query.type               | snapshot      | timetravel、incremental、snapshot |
  | START_OFFSET              | hoodie.datasource.streaming.startOffset    | earliest      | earliest、latest                  |
  | READ_PRE_COMBINE_FIELD    | hoodie.datasource.write.precombine.field   | ts            | 预聚合字段                        |
  | ENABLE_DATA_SKIPPING      | hoodie.enable.data.skipping                | false         | 启用数据索引                      |
  | ENABLE_HOODIE_FILE_INDEX  | hoodie.file.index.enable                   | true          | 文件索引                          |
  | SCHEMA_EVOLUTION_ENABLED  | hoodie.schema.on.read.enable               | false         | 模式演化                          |

- **spark写属性**：DataSourceWriteOptions

  | 属性                        | key                                                 | default_value           | 说明                                |
  | --------------------------- | --------------------------------------------------- | ----------------------- | ----------------------------------- |
  | HIVE_SYNC_MODE              | hoodie.datasource.hive_sync.mode                    |                         | hms、 jdbc、hiveql                  |
  | PARTITIONPATH_FIELD         | hoodie.datasource.write.partitionpath.field         |                         | 分区字段                            |
  | RECORDKEY_FIELD             | hoodie.datasource.write.recordkey.field             |                         | 主键                                |
  | TABLE_NAME                  | hoodie.datasource.write.table.name                  |                         | 表名                                |
  | ASYNC_CLUSTERING_ENABLE     | hoodie.clustering.async.enabled                     | false                   | 异步聚集                            |
  | INLINE_CLUSTERING_ENABLE    | hoodie.clustering.inline                            | false                   | 在线聚集                            |
  | ASYNC_COMPACT_ENABLE        | hoodie.datasource.compaction.async.enable           | true                    | 异步合并log文件                     |
  | HIVE_AUTO_CREATE_DATABASE   | hoodie.datasource.hive_sync.auto_create_database    | true                    | 自动建库                            |
  | HIVE_DATABASE               | hoodie.datasource.hive_sync.database                | default                 | 同步的hive库                        |
  | HIVE_SYNC_ENABLED           | hoodie.datasource.hive_sync.enable                  | false                   | 是否同步到hive                      |
  | METASTORE_URIS              | hoodie.datasource.hive_sync.metastore.uris          | thrift://localhost:9083 | hive元数据链接                      |
  | HIVE_PARTITION_FIELDS       | hoodie.datasource.hive_sync.partition_fields        |                         | hive分区                            |
  | HIVE_TABLE                  | hoodie.datasource.hive_sync.table                   | unknown                 | 同步的表                            |
  | INSERT_DROP_DUPS            | hoodie.datasource.write.insert.drop.duplicates      | false                   | 插入期间是否过滤                    |
  | OPERATION                   | hoodie.datasource.write.operation                   | upsert                  | insert、upsert、bulk_insert、delete |
  | PRECOMBINE_FIELD            | hoodie.datasource.write.precombine.field            | ts                      | 预聚合字段                          |
  | STREAMING_RETRY_CNT         | hoodie.datasource.write.streaming.retry.count       | 3                       | 重复次数                            |
  | STREAMING_RETRY_INTERVAL_MS | hoodie.datasource.write.streaming.retry.interval.ms | 2000                    | 重试间隔                            |
  | TABLE_TYPE                  | hoodie.datasource.write.table.type                  | COPY_ON_WRITE           | MERGE_ON_READ                       |
  | SQL_ENABLE_BULK_INSERT      | hoodie.sql.bulk.insert.enable                       | false                   | 开启bulk_insert                     |
  | SQL_INSERT_MODE             | hoodie.sql.insert.mode                              | upsert                  | insert、upsert、bulk_insert、delete |

#### 客户端属性

- **写属性**：HoodieWriteConfig

  | 属性                                    | key                                      | default_value | 说明           |
  | --------------------------------------- | ---------------------------------------- | ------------- | -------------- |
  | TBL_NAME                                | table_name                               |               | 表名           |
  | PRECOMBINE_FIELD_NAME                   | hoodie.datasource.write.precombine.field | ts            | 预聚合字段     |
  | BASE_PATH                               | hoodie.base.path                         |               |                |
  | MERGE_ALLOW_DUPLICATE_ON_INSERTS_ENABLE | hoodie.merge.allow.duplicate.on.inserts  | false         | 是否允许重复键 |

- **清理属性**：HoodieCleanConfig

  | 属性                           | key                                  | default_value | 说明                      |
  | ------------------------------ | ------------------------------------ | ------------- | ------------------------- |
  | ASYNC_CLEAN                    | hoodie.clean.async                   | false         | 异步清理                  |
  | CLEAN_MAX_COMMITS              | hoodie.clean.max.commits             | 1             | 间隔多少次提交            |
  | CLEAN_TRIGGER_STRATEGY         | hoodie.clean.trigger.strategy        | NUM_COMMITS   |                           |
  | CLEANER_COMMITS_RETAINED       | hoodie.cleaner.commits.retained      | 10            | 保留未清理的提交数        |
  | CLEANER_FILE_VERSIONS_RETAINED | hoodie.cleaner.fileversions.retained | 3             | KEEP_LATEST_FILE_VERSIONS |

- **布局属性**：HoodieLayoutConfig

- **内存属性**：HoodieMemoryConfig

- **归档属性**：HoodieArchivalConfig

- **元数据属性**：HoodieMetadataConfig

- **Check属性**：ConsistencyGuardConfig

- **文件系统属性**：FileSystemRetryConfig

- **聚集属性**：HoodieClusteringConfig

- **公共属性**：HoodieCommonConfig

- **引导属性**：HoodieBootstrapConfig

- **回调属性**：HoodieWriteCommitCallbackConfig

- **锁属性**：HoodieLockConfig

- **索引属性**：HoodieIndexConfig

#### 元数据属性

- **公共元数据属性**：HoodieSyncConfig
- **全局hive属性**：GlobalHiveSyncConfig
- **hive属性**：HiveSyncConfig

#### 监控属性

- **监控属性**：HoodieMetricsGraphiteConfig

#### 重载属性

- **重载属性**：HoodiePayloadConfig

>  [!note]
>
> 以上这些属性都可以在官网查找到：https://hudi.apache.org/docs/configurations
>
> 这里只列了一些可能会用到的属性

### 4、使用sql模式

使用spark-sql需要用到的属性也是上面列出来的，参照key

#### 建表

- 标准建表

  ```sql
  create table hudi_cow_pt_tbl (
    id bigint,
    name string,
    ts bigint,
    dt string,
    hh string
  ) using hudi
  tblproperties (
    type = 'cow',
    primaryKey = 'id',
    preCombineField = 'ts'
   )
  partitioned by (dt, hh)
  location '/tmp/hudi/hudi_cow_pt_tbl';
  ```

- 对已存在的表建表

  ```sql
  create table hudi_existing_tbl using hudi
  location '/tmp/hudi/hudi_existing_table';
  ```

- CTS建表

  ```sql
  -- CTAS: 创建非分区表
  create table hudi_ctas_cow_nonpcf_tbl
  using hudi
  tblproperties (primaryKey = 'id')
  as
  select 1 as id, 'a1' as name, 10 as price;
  
  -- CTAS: 创建分区表带属性
  create table hudi_ctas_cow_pt_tbl
  using hudi
  tblproperties (type = 'cow', primaryKey = 'id', preCombineField = 'ts')
  partitioned by (dt)
  as
  select 1 as id, 'a1' as name, 10 as price, 1000 as ts, '2021-12-01' as dt;
  
  -- 临时表
  create table parquet_mngd using parquet location 'file:///tmp/parquet_dataset/*.parquet';
  
  -- CTAS：导入数据到hudi表
  create table hudi_ctas_cow_pt_tbl2 using hudi location 'file:/tmp/hudi/hudi_tbl/' options (
    type = 'cow',
    primaryKey = 'id',
    preCombineField = 'ts'
   )
  partitioned by (datestr) as select * from parquet_mngd;
  ```

#### 插入数据

- 基本插入insert/upsert

  > [!note]
  >
  > 默认情况下，如果提供了preCombineKey, insert into写操作的类型是upsert，否则使用insert

  ```sql
  -- insert into non-partitioned table
  insert into hudi_cow_nonpcf_tbl select 1, 'a1', 20;
  insert into hudi_mor_tbl select 1, 'a1', 20, 1000;
  
  -- insert dynamic partition
  insert into hudi_cow_pt_tbl partition (dt, hh)
  select 1 as id, 'a1' as name, 1000 as ts, '2021-12-09' as dt, '10' as hh;
  
  -- insert static partition
  insert into hudi_cow_pt_tbl partition(dt = '2021-12-09', hh='11') select 2, 'a2', 1000;
  
  -- upsert mode for preCombineField-provided table
  insert into hudi_mor_tbl select 1, 'a1_1', 20, 1001;
  select id, name, price, ts from hudi_mor_tbl;
  1   a1_1    20.0    1001
  ```

- bulk_insert

  > [!note]
  >
  > 支持使用bulk_insert作为写操作的类型，只需要设置两个配置：**hoodie.sql.bulk.insert.enable**和**hoodie.sql.insert.mode**

  ```sql
  -- bulk_insert mode for preCombineField-provided table
  set hoodie.sql.bulk.insert.enable=true;
  set hoodie.sql.insert.mode=non-strict;
  
  insert into hudi_mor_tbl select 1, 'a1_2', 20, 1002;
  select id, name, price, ts from hudi_mor_tbl;
  1   a1_1    20.0    1001
  1   a1_2    20.0    1002
  ```

- 允许重复插入

  > [!note]
  >
  > 允许重复插入，则需要设置hoodie.merge.allow.duplicate.on.inserts和hoodie.datasource.write.operation属性

  ```sql
  set hoodie.merge.allow.duplicate.on.inserts = true;
  set hoodie.datasource.write.operation=insert;
  
  insert into hudi_mor_tbl select 1, 'a1_1', 20, 1001;
  select id, name, price, ts from hudi_mor_tbl;
  1   a1_1    20.0    1001
  1   a1_1    20.0    1001
  ```

- insert overwrite

  ```sql
  -- insert overwrite 非分区
  insert overwrite hudi_mor_tbl select 99, 'a99', 20.0, 900;
  insert overwrite hudi_cow_nonpcf_tbl select 99, 'a99', 20.0;
  
  -- insert overwrite 动态分区
  insert overwrite table hudi_cow_pt_tbl select 10, 'a10', 1100, '2021-12-09', '10';
  
  -- insert overwrite 静态分区
  insert overwrite hudi_cow_pt_tbl partition(dt = '2021-12-09', hh='12') select 13, 'a13', 1100;
  ```

  

#### 读取数据

- 快照查询

  ```sql
  select fare, begin_lon, begin_lat, ts from  hudi_trips_snapshot where fare > 20.0
  ```

- 时间旅行

  ```sql
  -- 测试表
  create table hudi_cow_pt_tbl (
    id bigint,
    name string,
    ts bigint,
    dt string,
    hh string
  ) using hudi
  tblproperties (
    type = 'cow',
    primaryKey = 'id',
    preCombineField = 'ts'
   )
  partitioned by (dt, hh)
  location '/tmp/hudi/hudi_cow_pt_tbl';
  
  -- 插入数据
  insert into hudi_cow_pt_tbl select 1, 'a0', 1000, '2021-12-09', '10';
  select * from hudi_cow_pt_tbl;
  
  -- 更新数据
  -- record id=1 changes `name`
  insert into hudi_cow_pt_tbl select 1, 'a1', 1001, '2021-12-09', '10';
  select * from hudi_cow_pt_tbl;
  
  -- 时间旅行,指定提交的时间戳
  -- time travel based on first commit time, assume `20220307091628793`
  select * from hudi_cow_pt_tbl timestamp as of '20220307091628793' where id = 1;
  
  -- 时间旅行,不同的时间格式
  -- time travel based on different timestamp formats
  select * from hudi_cow_pt_tbl timestamp as of '2022-03-07 09:16:28.100' where id = 1;
  select * from hudi_cow_pt_tbl timestamp as of '2022-03-08' where id = 1;
  ```

#### 更新数据

- 更新

  ```sql
  update hudi_mor_tbl set price = price * 2, ts = 1111 where id = 1;
  
  update hudi_cow_pt_tbl set name = 'a1_1', ts = 1001 where id = 1;
  
  -- 不使用主键更新(不建议)
  update hudi_cow_pt_tbl set ts = 1001 where name = 'a1';
  ```

  > [!note]
  >
  > 更新操作需要指定预聚合字段**preCombineField**。

- 合并

  案例1

  ```sql
  -- 源表
  create table merge_source (id int, name string, price double, ts bigint) using hudi
  tblproperties (primaryKey = 'id', preCombineField = 'ts');
  -- 插入数据
  insert into merge_source values (1, "old_a1", 22.22, 900), (2, "new_a2", 33.33, 2000), (3, "new_a3", 44.44, 2000);
  
  -- 合并到目标表, 关联键id, 关联上就更新, 没关联上就插入
  merge into hudi_mor_tbl as target
  using merge_source as source
  on target.id = source.id
  when matched then update set *
  when not matched then insert *
  ;
  ```

  案例2

  ```sql
  -- 源表
  create table merge_source2 (id int, name string, flag string, dt string, hh string) using parquet;
  -- 插入数据
  insert into merge_source2 values (1, "new_a1", 'update', '2021-12-09', '10'), (2, "new_a2", 'delete', '2021-12-09', '11'), (3, "new_a3", 'insert', '2021-12-09', '12');
  
  -- 合并到目标表, 首先对源表进行查询, 然后根据字段信息进行匹配更新或者插入
  merge into hudi_cow_pt_tbl as target
  using (
    select id, name, '1000' as ts, flag, dt, hh from merge_source2
  ) source
  on target.id = source.id
  when matched and flag != 'delete' then
   update set id = source.id, name = source.name, ts = source.ts, dt = source.dt, hh = source.hh
  when matched and flag = 'delete' then delete
  when not matched then
   insert (id, name, ts, dt, hh) values(source.id, source.name, source.ts, source.dt, source.hh)
  ;
  ```

#### 删除数据

```sql
delete from hudi_cow_nonpcf_tbl where uuid = 1;

delete from hudi_mor_tbl where id % 2 = 0;

-- 不使用主键进行删除(不建议)
delete from hudi_cow_pt_tbl where name = 'a1';
```

#### Alter Table

```sql
#改表名
--rename to:
ALTER TABLE hudi_cow_nonpcf_tbl RENAME TO hudi_cow_nonpcf_tbl2;

#加列
--add column:
ALTER TABLE hudi_cow_nonpcf_tbl2 add columns(remark string);

#改列
--change column:
ALTER TABLE hudi_cow_nonpcf_tbl2 change column uuid uuid bigint;

#设置属性
--set properties;
alter table hudi_cow_nonpcf_tbl2 set tblproperties (hoodie.keep.max.commits = '10');

--show partition:
show partitions hudi_cow_pt_tbl;

--drop partition：
alter table hudi_cow_pt_tbl drop partition (dt='2021-12-09', hh='10');

--show commit's info
call show_commits(table => 'test_hudi_table', limit => 10);
```







