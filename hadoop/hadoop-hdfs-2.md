# HDFS详解

## HDFS定义

HDFS（Hadoop Distributed File System），它是一个文件系统，用于存储文件，通过目录树来定位文件；

其次，它是分布式的，由很多服务器联合起来实现其功能，集群中的服务器有各自的角色。

HDFS只是分布式文件管理系统中的一种。

HDFS的使用场景：**适合一次写入，多次读出的场景**。一个文件经过创建、写入和关闭之后就不需要改变。

## HDFS优缺点

### 优点

- 高容错性：
  - 数据自动报错多个副本，通过增加副本的形式，提高容错性
  - 某一副本丢失后，它可以自动恢复
- 适合处理大数据
  - 数据规模：能够处理数据规模达到GB，TB，甚至PB级别的数据
  - 文件规模：能够处理百万规模以上的文件数量
- 可构建在廉价机器上，通过多副本机制，提高可靠性

### 缺点

- 不适合低延时数据访问，比如毫秒级的存储数据
- 无法高效的对大量小文件进行存储
  - 存储大量小文件，会占用NameNode大量的内存来存储文件目录和块信息
  - 小文件存储的寻址地址会超过读取时间
- 不支持并发写入，文件随机修改
  - 一个文件只能有一个写，不允许多个线程同时写
  - 仅支持数据追加（append），不支持文件的随机修改

## HDFS的组成架构

### 组成架构

![HDFS组成架构](hadoop-hdfs-2.assets/HDFS-1.png ':size=50%')

- NameNode：就是Master，它是管理者
  - 管理HDFS的名称空间
  - 配置副本策略
  - 管理数据块映射信息
  - 处理客户端读写请求
- DataNode：就是Slave，NameNode下达指令，DataNode执行实际的操作
  - 存储实际的数据块
  - 执行数据块的读写操作
- Client：就是客户端
  - 文件切分，文件上次HDFS时，Client将文件切分成一个一个的Block，然后进行上传
  - 与NameNode交互，获取文件的位置信息
  - 与DataNode交互，读取或者写入数据
  - Client提供一些命令来管理HDFS，比如NameNode格式化
  - Client提供一些命令来访问HDFS，比如对HDFS的增删查改
- Secondary NameNode：并非NameNode的热备，当NameNode挂掉的时候，它并不能马上替换NameNode并提供服务
  - 辅助NameNode，分担工作量
  - 在紧急情况下，可辅助恢复NameNode

### HDFS文件块

HDFS中的文件在物理机上是分块存储，块的大小可以通过配置参数（dfs.blocksize）来规定，默认大小在Hadoop2/3版本中是128M

- HDFS块的大小设置太小，会增加寻址时间，程序已知在找块的开始位置
- HDFS块的大小设置太大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需时间，导致程序处理这块数据时，会非常慢

HDFS块的大小设置主要取决于磁盘传输速率

## HDFS的Shell操作


