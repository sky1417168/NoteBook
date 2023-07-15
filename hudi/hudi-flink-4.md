# Hudi集成Flink

## 1、前提

需要使用hudi与flink相关的包，可以选择自己编译或者去maven仓库下载

使用的包名如下：

```bash
hudi-flink1.14-bundle-0.12.0.jar
```

## 2、flink-sql

参考官网教程：https://hudi.apache.org/docs/next/flink-quick-start-guide

官网的教程是启动flink集群做测试的，我是使用yarn，然后dinky上写sql测试，dinky对flink-sql的开发是比较友好的，客户端启动yarn-session再启动sql-client的方式略显笨重。

### 建表并插入数据

```sql
# 使用flink-表模式
set sql-client.execution.result-mode = tableau;

CREATE TABLE t1(
  uuid VARCHAR(20) PRIMARY KEY NOT ENFORCED,
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = '${path}',
  'table.type' = 'MERGE_ON_READ' -- this creates a MERGE_ON_READ table, by default is COPY_ON_WRITE
);

-- insert data using values
INSERT INTO t1 VALUES
  ('id1','Danny',23,TIMESTAMP '1970-01-01 00:00:01','par1'),
  ('id2','Stephen',33,TIMESTAMP '1970-01-01 00:00:02','par1'),
  ('id3','Julian',53,TIMESTAMP '1970-01-01 00:00:03','par2'),
  ('id4','Fabian',31,TIMESTAMP '1970-01-01 00:00:04','par2'),
  ('id5','Sophia',18,TIMESTAMP '1970-01-01 00:00:05','par3'),
  ('id6','Emma',20,TIMESTAMP '1970-01-01 00:00:06','par3'),
  ('id7','Bob',44,TIMESTAMP '1970-01-01 00:00:07','par4'),
  ('id8','Han',56,TIMESTAMP '1970-01-01 00:00:08','par4');
```

> [!note]
>
> flink-sql有几种模式可供选择：
>
> ​	1、sql-client.execution.result-mode = tableau，更接近传统的数据库，会将执行的结果以制表的形式直接打在屏幕之上
>
> ​	2、sql-client.execution.result-mode = changelog，不会实体化和可视化结果，而是由插入（+）和撤销（-）组成的持续查询产生结果流
>
> ​	3、sql-client.execution.result-mode = table，在内存中实体化结果，并将结果用规则的分页表格可视化展示出来

### 查询数据

```sql
select * from t1;
```







## 3、参数配置



## 4、问题



## 5、案例