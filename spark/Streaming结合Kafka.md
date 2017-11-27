# Spark2.11 两种流操作 + Kafka
Spark2.x 自从引入了 `Structured Streaming` 后，未来数据操作将逐步转化到 `DataFrame/DataSet`，以下将介绍 Spark2.x 如何与 `Kafka0.10+`整合
## Structured Streaming + Kafka
1. 引包
```
groupId = org.apache.spark
artifactId = spark-sql-kafka-0-10_2.11
version = 2.1.1
```
为了让更直观的展示包的依赖，以下是我的工程 sbt 文件
```
name := "spark-test"
version := "1.0"
scalaVersion := "2.11.7"
// https://mvnrepository.com/artifact/org.apache.spark/spark-core_2.11
libraryDependencies += "org.apache.spark" % "spark-core_2.11" % "2.1.1" % "provided"
// https://mvnrepository.com/artifact/org.apache.spark/spark-mllib_2.11
libraryDependencies += "org.apache.spark" % "spark-mllib_2.11" % "2.1.1" % "provided"
// https://mvnrepository.com/artifact/org.apache.spark/spark-streaming_2.11
libraryDependencies += "org.apache.spark" % "spark-streaming_2.11" % "2.1.1" % "provided"
// https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-client
libraryDependencies += "org.apache.hadoop" % "hadoop-client" % "2.7.3"
// https://mvnrepository.com/artifact/mysql/mysql-connector-java
libraryDependencies += "mysql" % "mysql-connector-java" % "5.1.38"
// https://mvnrepository.com/artifact/org.apache.kafka/kafka_2.11
libraryDependencies += "org.apache.kafka" % "kafka_2.11" % "0.10.2.1"
//libraryDependencies += "org.apache.spark" % "spark-streaming-kafka-0-10_2.11" % "2.1.1"
libraryDependencies += "org.apache.spark" % "spark-sql-kafka-0-10_2.11" % "2.1.1"
```
2. Structured Streaming 连接 Kafka
```
def main(args: Array[String]): Unit = {

    val spark = SparkSession
      .builder()
      .appName("Spark structured streaming Kafka example")
      //      .master("local[2]")
      .getOrCreate()

    val inputstream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "127.0.0.1:9092")
      .option("subscribe", "testss")
      .load()
    import spark.implicits._
    val query = inputstream.select($"key", $"value")
      .as[(String, String)].map(kv => kv._1 + " " + kv._2).as[String]
      .writeStream
      .outputMode("append")
      .format("console")
      .start()

    query.awaitTermination()
  }
```

流的元数据如下

| Column | Type |
|--------|--------|
|    key    |    binary    |
|    value    | binary    
|    topic    |    string    |
|    partition    |    int    |
|    offset    |    long    |
|    timestamp    |    long    |
|    timestampType    |    int    |

**可配参数**

| Option | value |   meaning |
|--------|--------|--------|
|    assign    |    json string {"topicA":[0,1],"topicB":[2,4]}    |   用于指定消费的 TopicPartitions，`assign`，`subscribe`，`subscribePattern` 是三种消费方式，只能同时指定一个   |
|    subscribe    | A comma-separated list of topics           |    用于指定要消费的 topic  |
|    subscribePattern    |    Java regex string    |   使用正则表达式匹配消费的 topic    |
|    kafka.bootstrap.servers    |    A comma-separated list of host:port    |    kafka brokers      |

**不能配置的参数**
- `group.id`: 对每个查询，kafka 自动创建一个唯一的 group
- `auto.offset.reset`: 可以通过 startingOffsets 指定，Structured Streaming 会对任何流数据维护 offset, 以保证承诺的 exactly once.
- `key.deserializer`: 在 DataFrame 上指定，默认 `ByteArrayDeserializer`
- `value.deserializer`: 在 DataFrame 上指定，默认 `ByteArrayDeserializer`
- `enable.auto.commit`:
- `interceptor.classes`:

## Stream + Kafka

1. 从最新offset开始消费

	```
	def main(args: Array[String]): Unit = {
		val kafkaParams = Map[String, Object](
		  "bootstrap.servers" -> "localhost:9092",
		  "key.deserializer" -> classOf[StringDeserializer],
		  "value.deserializer" -> classOf[StringDeserializer],
		  "group.id" -> "use_a_separate_group_id_for_each_stream",
		  "auto.offset.reset" -> "latest",
		  "enable.auto.commit" -> (false: java.lang.Boolean)
		)

		val ssc =new StreamingContext(OpContext.sc, Seconds(2))
		val topics = Array("test")
		val stream = KafkaUtils.createDirectStream[String, String](
		  ssc,
		  PreferConsistent,
		  Subscribe[String, String](topics, kafkaParams)
		)
		stream.foreachRDD(rdd=>{
		  val offsetRanges=rdd.asInstanceOf[HasOffsetRanges].offsetRanges
		  rdd.foreachPartition(iter=>{
			val o: OffsetRange = offsetRanges(TaskContext.get.partitionId)
			println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
		  })
		  stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
		})

	//    stream.map(record => (record.key, record.value)).print(1)
		ssc.start()
		ssc.awaitTermination()
	  }
	```

2. 从指定的offset开始消费

	```
	def main(args: Array[String]): Unit = {
	  val kafkaParams = Map[String, Object](
		"bootstrap.servers" -> "localhost:9092",
		"key.deserializer" -> classOf[StringDeserializer],
		"value.deserializer" -> classOf[StringDeserializer],
		"group.id" -> "use_a_separate_group_id_for_each_stream",
		//      "auto.offset.reset" -> "latest",
		"enable.auto.commit" -> (false: java.lang.Boolean)
	  )
	  val ssc = new StreamingContext(OpContext.sc, Seconds(2))
	  val fromOffsets = Map(new TopicPartition("test", 0) -> 1100449855L)
	  val stream = KafkaUtils.createDirectStream[String, String](
		ssc,
		PreferConsistent,
		Assign[String, String](fromOffsets.keys.toList, kafkaParams, fromOffsets)
	  )

	  stream.foreachRDD(rdd => {
		val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
		for (o <- offsetRanges) {
		  println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
		}
		stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
	  })

	  //    stream.map(record => (record.key, record.value)).print(1)
	  ssc.start()
	  ssc.awaitTermination()
	}
	```
