# DataFrame/DataSet 创建

* 读文件接口
```
import org.apache.spark.sql.SparkSession
val spark = SparkSession
  .builder()
  .appName("Spark SQL basic example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()
// For implicit conversions like converting RDDs to DataFrames
import spark.implicits._
val df=spark.read.xxx
```
**[`DataFrame/DataSet` 读取数据源文档](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameReader) <br><br>`spark.read` 返回 `DataFrameReader` <br><br> `spark.readStream` 返回 `DataStreamReader`<br><br>[后续读文件操作雷同，可以参考作者的 `Structured Streaming` 文章](http://ishare.58corp.com/index.php/topic/show/744)**

* `RDD` 转换成 `DataFrame/DataSet`
  - 方式1：已知元数据
	```
	val peopleDF = spark.sparkContext
	  .textFile("examples/src/main/resources/people.txt")
	  .map(_.split(","))
	  .map(attributes => Person(attributes(0), attributes(1).trim.toInt))
	  .toDF()/toDS
	```
  - 方式2：未知元数据
	```
	val schemaString = "name age"
	// Generate the schema based on the string of schema
	val fields = schemaString.split(" ")
	  .map(fieldName => StructField(fieldName, StringType, nullable = true))
	val schema = StructType(fields)
	// Convert records of the RDD (people) to Rows
	val rowRDD = peopleRDD
	  .map(_.split(","))
	  .map(attributes => Row(attributes(0), attributes(1).trim))
	```
