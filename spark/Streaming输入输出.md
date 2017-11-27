# Structured Streaming 输入输出
## 输入
`SparkSession.readStream()` 返回一个 `DataStreamReader` 接口对象，可以通过该对象对输入源进行参数配置，最后返回DataFrame/DataSet对象。
## 输入源有三种
* `File` : `csv`,`json`,`text`,`textFile` 等
```
val csvDF = spark
  .readStream
  .option("sep", ";")
  .schema(userSchema)
  .csv("/path/to/directory")
```
* `Kafka` ：
```
val inputstream = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "127.0.0.1:9092")
  .option("subscribe", "testss")
  .load()
```
* `Socket` :
```
val socketDF = spark
  .readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()
```
**[具体输入配置参考创建 ](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.streaming.DataStreamReader)**

## 输出模式
 * `Append` 模式(默认)：只有新加入 `Result Table` 行才会输出，保证每行只会往输出端输出一次，当操作为 `select`, `where`, `map`, `flatMap`, `filter`, `join` 等才支持 `append` 模式。
 * `Complete` 模式：每次会把整个 `Result Table` 输出，所以只支持聚合操作。
 * `Update` 模式：只有更新的数据才会输出到输出端(内存中维护了上次触发后的结果)。
不同的流查询操作支持不同的输出模式，如下表所示：

|查询类型|支持的模式|原因|
|----------|--------------------|-------------------|
|非聚合操作|`Append`<br>`Update`|`Complete`模式不支持是因为需要在 `Result Table` 中维护所有数据，这是不太现实的|
|基于watermark的窗口聚合操作|`Append`<br>`Update`<br>`Complete`|`Append`当确定不会更新窗口时，将会输出该窗口的数据并删除，保证每个窗口的数据只会输出一次 <br> `Update` 删除不再更新的时间窗口，每次触发聚合操作时，输出更新的窗口 <br> `Complete` 不删除任何数据，在 `Result Table` 中保留所有数据，每次触发操作输出所有窗口数据
|其他聚合操作|`Update`<br>`Complete`| `Update` 每次触发聚合操作时，输出更新的窗口 <br> `Complete` 不删除任何数据，在 `Result Table` 中保留所有数据，每次触发操作输出所有窗口数据  <br>`Append` 聚合操作用于更新分组，这与 `Append` 的语义相违背|

## 输出端
* `File 输出` - 指定输出的目录(输出模式：`Append`)
```
writeStream
    .format("parquet")        // can be "orc", "json", "csv", etc.
    .option("path", "path/to/destination/dir")
    .start()
```
* `Foreach` 输出 - 实现自定义(`Append`,`Update`,`Complete`)
```
writeStream
    .foreach(...)
    .start()
```
* `Console` 输出 - 用于调试(`Append`,`Update`,`Complete`)
```
writeStream
    .format("console")
    .start()
```
* `Memory` 输出(`Append`,`Complete`)
```
writeStream
    .format("memory")
    .queryName("tableName")
    .start()
```

### `Foreach` 实现自定义输出
```
val query = wordCounts.writeStream.trigger(ProcessingTime(5.seconds))
      .outputMode("complete")
      .foreach(new ForeachWriter[Row] {

      var fileWriter: FileWriter = _

      override def process(value: Row): Unit = {
        fileWriter.append(value.toSeq.mkString(","))
      }

      override def close(errorOrNull: Throwable): Unit = {
        fileWriter.close()
      }

      override def open(partitionId: Long, version: Long): Boolean = {
        FileUtils.forceMkdir(new File(s"/tmp/example/${partitionId}"))
        fileWriter = new FileWriter(new File(s"/tmp/example/${partitionId}/temp"))
        true
      }
    }).start()
```


