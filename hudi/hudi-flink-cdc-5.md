# Flink CDC入湖

### 环境

- flink 1.14.6
- hudi 0.12.0
- mysql 5.7.0  
- dinky 0.7.2
- flink-connector-mysql-cdc: 2.2.1
- hive 3.1.2
- hadoop 3.3.4
- spark 3.3.1

### 前提准备

- mysql开启binlog，开启GTID高可用（可选）

  [参考flink-cdc mysql 文档: https://ververica.github.io/flink-cdc-connectors/release-2.2/content/connectors/mysql-cdc%28ZH%29.html](https://ververica.github.io/flink-cdc-connectors/release-2.2/content/connectors/mysql-cdc(ZH).html)

- 编译flink1.14版本的dinky，并部署，配置好jar包到plugins目录，注意修改启动脚本里flink版本为1.14

- flink lib目录配置jar 和 dinky的plugins配置的jar包完全一样即可

  ```java
  commons-cli-1.4.jar
  flink-csv-1.14.6.jar
  flink-dist_2.12-1.14.6.jar
  flink-faker-0.3.0.jar
  flink-json-1.14.6.jar
  flink-shaded-hadoop-3-uber-3.1.1.7.2.9.0-173-9.0.jar
  flink-shaded-zookeeper-3.4.14.jar
  flink-sql-connector-hive-3.1.2_2.12-1.14.6.jar
  flink-sql-connector-jdbc-2.1.3.jar
  flink-sql-connector-kafka_2.12-1.14.6.jar
  flink-sql-connector-mysql-cdc-2.2.1.jar
  flink-sql-connector-oracle-cdc-2.2.1.jar
  flink-sql-connector-sqlserver-cdc-2.2.1.jar
  flink-table_2.12-1.14.6.jar
  guava-27.0-jre.jar
  hadoop-mapreduce-client-core-3.3.4.jar
  hudi-flink1.14-bundle-0.12.0.jar
  kafka-clients-2.4.1.jar
  log4j-1.2-api-2.17.1.jar
  log4j-api-2.17.1.jar
  log4j-core-2.17.1.jar
  log4j-slf4j-impl-2.17.1.jar
  mysql-connector-java-8.0.16.jar
  ```

- 编译hudi，使用flink1.14版本编译

  ```bash
  mvn clean package -DskipTests -Dspark3.2 -Dflink1.14 -Dscala-2.12 -Dhadoop.version=3.1.3 -Pflink-bundle-shade-hive3 -Davro.version=1.11.1
  ```

  > [!tip]
  >
  > 这里是完整的编译指令，可根据不同情况选择不一样

### 实操

- hive catalog

  ```sql
  CREATE CATALOG hive WITH ( 
      'type' = 'hive',
      'default-database' = 'default',
      'hive-conf-dir' = '/data/service/hive/conf/', --hive配置文件
      'hadoop-conf-dir'='/data/service/dlink-0.7.2/config/hadoopConf/' --hadoop配置文件，配了环境变量则不需要。
  );
  ```

  > [!note]
  >
  > 使用hive catalog的好处在于可以持久化flink的connector映射表，hive的配置文件路径引用hive的即可，hadoop的配置文件路径只需要常用的4个hdfs-site.xml，yarn-site.xml，core-site.xml，mapred-site.xml，这里是新建目录复制过来的

- 建表mysql connector

  ```sql
  USE CATALOG hive;
  
  use hudi_flink_cdc;
  
  drop table if EXISTS mysql_source;
  create table mysql_source(id int not null,
   ehlogid int,
   objid int,
   `type` int,
   ctmid int,
   hruid int,
   coid int,
   divid int,
   checktype string,
   checkqueue string,
   dbid int,
   rejectdata string,
   operator string,
   create_at string,
   update_at string,
   updatedata string,
   rejecttype string,
   oprtype string,
   primary key(id)  NOT ENFORCED)
   with ('connector' = 'mysql-cdc',
   'hostname' = 'xxx.xxx.xxx.xxx',
   'port' = '3306',
   'username' = 'xxx',
   'password' = 'xxx',
   'database-name' = 'xxx',
   'table-name' = 'xxxx',
   'scan.startup.mode' = 'latest-offset' 
   );
  ```

- hudi 建表 connector

  ```sql
  USE CATALOG hive;
  
  use hudi_flink_cdc;
  
  drop table if EXISTS hudi_sink;
  CREATE TABLE hudi_sink
  (
   id bigint ,
   ehlogid int,
   objid int,
   `type` int,
   ctmid int,
   hruid int,
   coid int,
   divid int,
   checktype string,
   checkqueue string,
   dbid int,
   rejectdata string,
   operator string,
   create_at string,
   update_at string,
   updatedata string,
   rejecttype string,
   oprtype string,
    primary key(id)  NOT ENFORCED
  ) COMMENT 'hudi_table'
  WITH (
  'connector' = 'hudi',
  'path' = '/bigdata/hudi/flink/hudi_sink', -- 路径会自动创建
  'hoodie.datasource.write.recordkey.field' = 'id', -- 主键
  'write.precombine.field' = 'update_at', -- 相同的键值时，取此字段最大值，默认ts字段
  'read.streaming.skip_compaction' = 'true', -- 避免重复消费问题
  -- 'write.bucket_assign.tasks' = '8', -- 并发写的 bucekt 数
  'write.tasks' = '24',
  'compaction.tasks' = '24',
  'write.operation' = 'insert', -- UPSERT（插入更新）\INSERT（插入）\BULK_INSERT（批插入）（upsert性能会低些，不适合埋点上报）
  -- 'write.rate.limit' = '20000', -- 限制每秒多少条
  'table.type' = 'MERGE_ON_READ', -- 默认COPY_ON_WRITE ，
  'compaction.async.enabled' = 'true', -- 在线压缩
  'compaction.trigger.strategy' = 'num_or_time', -- 按次数压缩
  'compaction.delta_commits' = '20', -- 默认为5
  'compaction.delta_seconds' = '60', -- 默认为1小时
  'hive_sync.enable' = 'true', -- 启用hive同步
  'hive_sync.mode' = 'hms', -- 启用hive hms同步，默认jdbc
  'hive_sync.metastore.uris' = 'thrift://xxx.xxx.xxx.xxx:9083', -- required, metastore的端口
  -- 'hive_sync.jdbc_url' = 'jdbc:hive2://192.168.74.182:10000', -- required, hiveServer地址
  'hive_sync.table' = 'hudi_sink', -- required, hive 新建的表名 会自动同步hudi的表结构和数据到hive
  'hive_sync.db' = 'hudi_flink_cdc' -- required, hive 新建的数据库名
  -- 'hive_sync.username' = 'hive', -- required, HMS 用户名
  -- 'hive_sync.password' = '123456', -- required, HMS 密码
  -- 'hive_sync.skip_ro_suffix' = 'true' -- 去除ro后缀
  );
  ```

- 建立读写任务

  ```sql
  USE CATALOG hive;
  
  use hudi_flink_cdc;
  
  
  SET execution.checkpointing.interval = 60000;
  
  SET execution.checkpointing.mode='EXACTLY_ONCE';
  
  insert into hudi_sink select * from mysql_source;
  ```

- 在dinky上配置fink per-job任务，设置好参数，启动即可

  ![flink-webui](img/hudi/image-20230723100601195.png)

- 然后可开启spark-sql客户端读取数据，也可以继续使用flink读取继续下一步处理

  ```bash
  spark-sql   --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer'   --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog'   --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension'    --master yarn --queue sd_queue
  ```

  ```sql
  use  hudi_flink_cdc;
  show tables;
  # 此时可以看到三张表
  # hudi_sink
  # hudi_sink_ro
  # hudi_sink_rt
  REFRESH TABLE hudi_sink_rt;
  select count(1) from hudi_sink_rt;
  ```

  >  [!note]
  >
  > 可以看到三张表，分别是hudi_sink表（不能查，这是flink映射表，只是持久化的，表里没字段），hudi_sink_ro 是hudi的读优化查询表，只能查到最新一次compaction之后的数据（实际上是查parquet文件，所以快），hudi_sink_rt 是快照查询，可以查到当前最新的所有数据，但是查询较慢，会有延迟，查询前需要刷新表，不然有可能会报文件找不到的错误

### 可能遇到的问题

1. spark-sql查询mor表报parwuet文件找不到

   ```sql
   REFRESH TABLE hudi_sink_rt;
   ```

   > [!note]
   >
   > 刷新表即可

2. 没有生成ro，rt表

   hadoop-mapreduce-client-core-3.1.3.jar

   flink-sql-connector-hive-3.1.2_2.12-1.14.6.jar

   > [!note]
   >
   > 首先lib目录里必须要有这两个包
   >
   > 然后删除flink-sql-connector-hive包里com.google目录即可

3. flink写到hudi时compaction失败，报以下错误

   ```bash
   Caused by: java.lang.NoSuchMethodError: org.apache.hudi.org.apache.avro.specific.SpecificRecordBuilderBase.<init>(Lorg/apache/hudi/org/apache/avro/Schema;Lorg/apache/hudi/org/apache/avro/specific/SpecificData;)V
   ```

   编译hudi时：修改为以下代码

   ```xml
   // \hudi-0.12.0\packaging\hudi-flink-bundle\pom.xml
   <dependency>
     <groupId>org.apache.avro</groupId>
     <artifactId>avro</artifactId>
     <!-- Override the version to be same with Flink avro -->
     <version>${avro.version}</version>
     <scope>compile</scope>
   </dependency>
   ```

4. flink做checkpoint错误，报以下错误

   ```bash
   [Flink-DispatcherRestEndpoint-thread-4] ERROR org.apache.flink.runtime.rest.handler.job.checkpoints.CheckpointingStatisticsHandler - Unhandled exception.
   java.lang.NoSuchMethodError: org.apache.commons.math3.stat.descriptive.rank.Percentile.withNaNStrategy(Lorg/apache/commons/math3/stat/ranking/NaNStrategy;)Lorg/apache/commons/math3/stat/descriptive/rank/Percentile;
   ```

   > [!tip]
   >
   > flink-shaded-hadoop-3-uber-3.1.1.7.2.9.0-173-9.0.jar 这个包版本的问题，换成最新版本即可
   >
   > commons-cli-1.4.jar 这个包不放会报启动yarn-session错误
   
   

