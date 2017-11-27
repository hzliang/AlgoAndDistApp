# DataFrame/DataSet 操作
`Databricks` 不止一次提到过希望未来在编写 `Spark` 应用程序过程中，对于结构化/半结构化数据，使用 `Datasets`(`DataFrame` 的扩展) 来代替 `RDD` 操作，这主要源于 `Datasets` 以下几个方面：
 * 充分利用了 `Catalyst` 编译优化器 和 `Tungsten` 执行引擎优化程序
 * 程序运行速度更快，以原始的二进制的方式进行某些操作
 * 序列化/反序列化速度更快，使用 Tungsten 序列化方式，减少网络传输
 * 缓存数据的内存消耗更少
 * 统一接口等
`Encoder` 编码器负责在表结构(`Datasets`)和 `JVM` 对象(`RDD`)之间转换。
## 操作1：
将 `DataFrame/DataSet` 映射到一张表中，然后使用 `Sql` 文档提供的函数进行操作 [`Spark-Sql-Functions` 文档](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$)

*`Sql` 中的方法参数分两种*
1. `String` 类型和 `Column` 类型的列名重载方法
2. `Column` 类型的列名方法

如下所示：
```
def min(e: Column): Column
def min(columnName: String): Column
def abs(e: Column): Column
```
**对于 `String` 类型的列名，我们可以先将 `DataFrame` 映射到一种表中，然后直接写 `Sql` 语句进行查询操作**
 ```
import spark.implicits._
val df = spark.readStream.text("hdfs://localhost:9000/names/yob1884.txt")
df.createGlobalTempView("people")
//value 为列名
spark.sql("select * from global_temp.people").show()
spark.sql("select approx_count_distinct(value,0.05) from global_temp.people" ).show()
spark.sql("select min(value) from global_temp.people").show()
 ```
**对于 `Column` 类型的列名，我们只能在 `DataFrame` 上调用 `select` 方法进行操作**
```
val spark = SparkSession
  .builder()
  .appName("Spark structured Steaming our output example")
  .getOrCreate()

import spark.implicits._
val df = spark.readStream
  .option("maxFilesPerTrigger", "1")
  .textFile("hdfs://localhost:9000/test")

val query = df.map(_.toString().split(","))
  .map(p => Person(p(0), p(1), Integer.parseInt(p(2))))
  .select($"name", $"age")
  .where("age>50")

import org.apache.spark.sql.functions._
val testDF = query.select(min($"age"))
```
## 操作2：
将 `DataFrame/DataSet` 转换成 `DataSet`，使用 `DataSet` 提供的函数进行操作[`DataSet` 操作文档](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset)
```
    //DataSet group By
    query.groupBy($"age").count()
    //sql group by
    spark.sql("select * from global_temp.people group by value")
```
