# Hudi编译

### 第一步：下载源码

[hudi官网下载地址](https://hudi.apache.org/releases/download)

> [!note] 本次编译使用版本**0.12.0**

### 第二步：手动安装Kafka依赖

#### 先下载一下jar包

- common-config-5.3.4.jar

- common-utils-5.3.4.jar

- kafka-avro-serializer-5.3.4.jar

- kafka-schema-registry-client-5.3.4.jar

#### 再安装到maven仓库

```bash
mvn install:install-file -DgroupId=io.confluent -DartifactId=common-config -Dversion=5.3.4 -Dpackaging=jar -Dfile=./common-config-5.3.4.jar

mvn install:install-file -DgroupId=io.confluent -DartifactId=common-utils -Dversion=5.3.4 -Dpackaging=jar -Dfile=./common-utils-5.3.4.jar

mvn install:install-file -DgroupId=io.confluent -DartifactId=kafka-avro-serializer -Dversion=5.3.4 -Dpackaging=jar -Dfile=./kafka-avro-serializer-5.3.4.jar

mvn install:install-file -DgroupId=io.confluent -DartifactId=kafka-schema-registry-client -Dversion=5.3.4 -Dpackaging=jar -Dfile=./kafka-schema-registry-client-5.3.4.jar
```

### 第三步：编译

指定版本编译：

- flink1.14
- scala2.12
- hadoop3.1.3
- hive-3
- spark3.2

```bash
mvn clean package -DskipTests -Dspark3.2 -Dflink1.14 -Dscala-2.12 -Dhadoop.version=3.1.3 -Pflink-bundle-shade-hive3
```

指定avro版本编译

- avro1.11.1

```bash
mvn clean package -DskipTests -Dspark3.3 -Dflink1.14 -Dscala-2.12 -Dhadoop.version=3.3.4 -Pflink-bundle-shade-hive3 -Davro.version=1.11.1
```

### 编译问题：

> [!tip]
>
> 以下问题本人遇到，如果遇到可采用对应方案解决。

#### 问题01：使用spark写数据第11次触发报错

```bash
java.lang.NoSuchMethodError:org.apache.hadoop.hdfs.client.HdfsDataInputStream.getReadStatistics()
```

解决方案：

https://github.com/apache/hudi/issues/5765

[FAQs | Apache Hudi](https://hudi.apache.org/docs/faq/#how-can-i-resolve-the-nosuchmethoderror-from-hbase-when-using-hudi-with-metadata-table-on-hdfs)

- 下载hbase源码 并切换到2.4.9分支

  ```bash
  git clone https://github.com/apache/hbase.git
  git checkout rel/2.4.9
  ```

- 编译hbase 2.4.9 对应 hadoop3

  这里在windows的命令行下编译回报bash的错误，使用git bash窗口编译

  ```bash
  mvn clean install -Denforcer.skip  -DskipTests -Dhadoop.profile=3.0 -Psite-install-step
  ```

- 替换maven仓库的hbase-server目录下的包

  ```
  hbase-server-2.4.9.jar
  hbase-server-2.4.9-tests.jar
  ```

- 再次编译hudi

#### 问题02：flink写数据avro报错

```bash
Caused by: java.lang.NoSuchMethodError: org.apache.hudi.org.apache.avro.specific.SpecificRecordBuilderBase.<init>(Lorg/apache/hudi/org/apache/avro/Schema;Lorg/apache/hudi/org/apache/avro/specific/SpecificData;)V
```

解决方案：

[[HUDI-4806\] Use Avro version from the root pom for Flink bundle by CTTY · Pull Request #6628 · apache/hudi (github.com)](https://github.com/apache/hudi/pull/6628)

```xml
//路径 \hudi-0.12.0\packaging\hudi-flink-bundle\pom.xml
<dependency>
  <groupId>org.apache.avro</groupId>
  <artifactId>avro</artifactId>
  <!-- Override the version to be same with Flink avro -->
  <version>${avro.version}</version>
  <scope>compile</scope>
</dependency>
```







