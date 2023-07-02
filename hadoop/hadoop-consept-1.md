# Hadoop基本概念

## Hadoop是什么

- Hadoop是由Apache基金会开发的**分布式系统基础架构**

- 主要解决海量数据的**存储**和海量数据的**分析计算**问题

- Google是Hadoop的思想之源：
  - GFS  				---> HDFS

  - Map-Reduce  ---> MR

  - BigTable         ---> HBase

## Hadoop优势

- 高可用性：Hadoop底层维护多个数据副本，所以即使Hadoop某个计算元素或者存储出现故障，也不会导致数据的丢失

- 高扩展性：在集群间分配任务数据，可方便的扩展节点

- 高效性：在Map-Reduce的思想下，Hadoop是并行工作的，以加快任务处理速度

- 高容错性：能够自动的将失败的任务重新分配

## Hadoop组成

### HDFS

Hadoop Distributed FileSystem，简称HDFS，是一个分布式文件系统

- NameNode：存储文件的**元数据**，如文件名，文件目录结构，文件属性（生成时间，副本数，文件权限），以及每个文件的块列表和块所在的DataNode等

- DataNode：本地文件系统**存储文件块数据**，以及**块数据的校验和**

- Secondary NameNode：每隔一段时间对NameNode元数据**备份**

### YARN

YetAnother Resource Negotiator，简称YARN，另一种资源协调者，是Hadoop的资源管理器

- ResourceManager：整个集群资源（内存，CPU等）的管理者

- NodeManager：单个阶段服务器资源的管理者

- ApplicationMaster：单个任务运行的资源管理者

- Container：容器，相当于一台独立的服务器，里面封装了任务运行所需的资源，如内存，CPU，磁盘，网络等

![YARN](hadoop-consept-1.assets/YARN-1.png ':size=50%')

### MapReduce

MapReduce将计算过程分为两个阶段：Map和Reduce

- Map阶段并行处理输入数据
- Reduce阶段对Map结果进行汇总

![MapReduce](hadoop-consept-1.assets/MR-2.png ':size=70%')

### 相互协调工作

![三者协调工作](hadoop-consept-1.assets/三者协调工作-3.png ':size=60%')

> [!note]
>
> 客户端Client提交任务到资源管理器（ResourceManager）
>
> 资源管理器接收到任务之后去NodeManager节点开启任务（ApplicationMaster）
>
> ApplicationMaster向ResourceManager申请资源，若有资源ApplicationMaster负责开启任务即MapTask
>
> 开始分析任务，每个map独立工作，各自负责检索各自对应的DataNode，将结果记录到HDFS
>
> DataNode负责存储，NameNode负责记录，2NN负责备份部分数据

