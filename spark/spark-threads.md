# **spark多线程**

## **1、背景**

数仓建设过程中，报表是一张张开发出来的，随着时间的推移，有些报表难免要补上之前的数据，例如，23年2月20开发了某张报表，那么一般需要补上近期一个月的数据，有些业务不一样，可能需要补上当年的，甚至是去年的前年的数据。

如果你使用hive的mr引擎去补数据，一旦你的宽表涉及表比较多，join比较多，按天跑都要花上不少时间，那么补数据就是一个难受的活，就一个字，慢！

也许是hive的版本过低，但是你又不能做主升级hive，版本低可能导致hive on spark 效率也是十分低下，当然也有可能是版本不兼容导致。

那么怎么才能迅速的补完数据，解放双手呢？经我测试，spark-sql可以做到，将你的sql丢入到spark-sql里跑，申请更多的资源，充分利用集群，那么很快就能解决补数据的问题。

但是，问题又来了，我一次要补一年，两年的数据，我一次全放进去跑，负担太大，但是我一个月一个月的话，我又要操作太多次，十分繁琐（ps: 讲道理也还好），那么有什么办法可以解决呢？

还真有！那就是spark-sql多线程。

## **2、需求**

开始撸代码之前，要搞清楚我们的需求是什么。

- 输入sql文件路径
- 输入开始日期，结束日期，自动替换sql里的日期
- 按月并发跑任务

## **3、代码**

### **3.1、参数设置**

```java
val conf = new SparkConf()
.set("hive.exec.dynamic.partition", "true")
.set("hive.exec.dynamic.partition.mode", "nonstrict")
.set("set hive.auto.convert.join", "true")
.set("spark.debug.maxToStringFields", "100")
.set("spark.sql.adaptive.enabled","true")
```

主要用于开启动态分区，优化执行

### **3.2、解析日期**

目的是根据输入日期返回按月切分的数组

```java
def get_month_date_list(start:String,end:String): Array[(String, String)] ={
    val dateFormat: SimpleDateFormat = new SimpleDateFormat("yyyyMMdd")
    var date_arr = Array[(String, String)]()

    val start_date = dateFormat.parse(start)
    val end_date = dateFormat.parse(end)

    val start_calendar = Calendar.getInstance(TimeZone.getTimeZone("Asia/Shanghai"))
    val end_calendar = Calendar.getInstance(TimeZone.getTimeZone("Asia/Shanghai"))

    start_calendar.setTime(start_date)
    end_calendar.setTime(end_date)

    val first_month_start = dateFormat.format(start_calendar.getTime)
    start_calendar.add(Calendar.MONTH,1)
    start_calendar.set(Calendar.DAY_OF_MONTH,0)

    val first_month_end = dateFormat.format(start_calendar.getTime)

    date_arr = date_arr :+ (first_month_start,first_month_end)

    while ((start_calendar.get(Calendar.MONTH) + 1 < end_calendar.get(Calendar.MONTH))
           || start_calendar.get(Calendar.YEAR) < end_calendar.get(Calendar.YEAR)){
        start_calendar.add(Calendar.MONTH,1)
        start_calendar.set(Calendar.DAY_OF_MONTH,1)
        val month_start = dateFormat.format(start_calendar.getTime)
        start_calendar.add(Calendar.MONTH,1)
        start_calendar.set(Calendar.DAY_OF_MONTH,0)
        val month_end = dateFormat.format(start_calendar.getTime)

        date_arr = date_arr :+  (month_start,month_end)
    }

    date_arr

}
```

### **3.3、解析sql**

由于spark运行在hadoop集群之上，所以sql文件放在hdfs上便于读取，不容易出错，加之一些基本判断处理，返回符合格式的sql字符串

```java
  def parse_sql_by_date(sql_file_path:String): String ={

    info("sql路径:\n" + sql_file_path)

    val sql_rdd = readText(sql_file_path)

    val sql_list = sql_rdd.collect()
    val builder = new StringBuilder()

    for (text <- sql_list) {
      builder.append(text).append("\n")
    }

    val sql_text = builder.toString.replace("$","")

    info("sql文件:\n" + sql_text)

    if (!sql_text.contains("date_start")){
      throw new Exception("set sql like dt >= date_start")
    }
    if (!sql_text.contains("date_end")){
      throw new Exception("set sql like dt <= date_end")
    }

    sql_text
  }
```

### **3.4、开启多线程**

需要注意的点是线程池的数量不要太多了

太多会导致同时启动的spark任务太多，从而加大namenode的压力，因为读文件频繁

每一个线程相当于开启一个spark任务，根据你目前的集群承受能力酌情设置

再者就是要主要需要设置一个返回值来保证线程堵塞，从而driver可以等到所有程序结束再结束，不然driver会一直挂在那，虽然程序已经执行完毕。

```java
object LoadDataMonth {

  private val spark: SparkSession = EnvUtil.getSparkSession

  def process(sql_file_path: String, date_start: String, date_end: String): Unit = {

    info("开始解析sql\n")
    info("date_start: " + date_start + "\n")
    info("date_end: " + date_end + "\n")


    val sql_text: String = parse_sql_by_date(sql_file_path)

    val date_list = get_month_date_list(date_start, date_end)

    val nums_threads = date_list.length

    info("nums_threads: " + nums_threads)

    val executors = Executors.newFixedThreadPool(10)
    var result_list = List[Future[String]]()


    for (elem <- date_list) {
      val task = executors.submit(new Callable[String] {
        override def call(): String = {
          info("sql执行...: " + elem._1 + "----" + elem._2)

          val regex_start = "date_start".r
          val regex_end = "date_end".r

          val sql_text_1 = regex_start.replaceAllIn(sql_text,elem._1)
          val sql_text_2 = regex_end.replaceAllIn(sql_text_1,elem._2)

          info("sql解析: \n" + sql_text_2)

          for (text <- sql_text_2.split(";")) {

            info("sql_text: \n" + text)

            val res = spark.sql(text)

            res.show(1)
          }

          "success"
        }
      })

      result_list = result_list :+ task
    }

    for (elem <- result_list) {
      val res = elem.get()
      println(res)
    }

    executors.shutdown()


  }
}
```