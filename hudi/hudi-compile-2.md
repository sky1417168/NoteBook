# Hudi编译

### 第一步：下载源码

[hudi官网下载地址](https://hudi.apache.org/releases/download)

> [!note] 本次编译使用最新版本**0.13.1**

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







