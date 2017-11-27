# Structured Streaming 之窗口事件时间聚合操作
`Spark Streaming` 中 `Exactly Once` 指的是：
* 每条数据**被处理**且**仅被一个** `batch` 处理，**不丢，不重**
* 输出端文件系统保证幂等关系

`Structured Streaming` 返回的是 `DataFrame/DataSet`，我们可以对其应用各种操作 - 从无类型，类似 SQL 的操作（例如 `select`，`where`，`groupBy`）到类型化的 RDD 类操作（例如 `map`，`filter`，`flatMap`）。

## 基本操作：选择，投影，聚合
```
case class DeviceData(device: String, deviceType: String,
   signal: Double, time: DateTime)

val df: DataFrame = ... // streaming DataFrame with IOT device data with schema { device: string, deviceType: string, signal: double, time: string }
val ds: Dataset[DeviceData] = df.as[DeviceData]    // streaming Dataset with IOT device data

// Select the devices which have signal more than 10
df.select("device").where("signal > 10")      // using untyped APIs   
ds.filter(_.signal > 10).map(_.device)         // using typed APIs

// Running count of the number of updates for each device type
df.groupBy("deviceType").count()                          // using untyped API

// Running average signal for each device type
import org.apache.spark.sql.expressions.scalalang.typed
ds.groupByKey(_.deviceType).agg(typed.avg(_.signal))    // using typed API
```
## 不支持的操作：
但是，不是所有适用于静态 `DataFrames/DataSet` 的操作在流式 `DataFrames/DataSet` 中受支持。从 Spark 2.0 开始，一些不受支持的操作如下：

* 在流 `DataFrame/DataSet` 上还不支持多个流聚集（即，流 DF 上的聚合链）。
* 不支持 `limit` 和 `take(N)`
* 不支持 `Distinct`
* `sort` 操作仅在聚合后在完整输出模式下支持
* 流和静态流的外连接支持是有条件的：
  * 不支持带有流 `DataSet` 的完全外连接
  * 不支持右侧的流的左外连接
  * 不支持左侧的流的右外部联接
* 不支持两个流之间的任何 `join`
* 此外，还有一些方法不能用于流`DataSet`，它们是将立即运行查询并返回结果的操作，这对流`DataSet`没有意义。相反，这些功能可以通过显式地启动流查询来完成。
* `count()` - 无法从流 `DataSet` 返回单个计数。
相反，使用 `ds.groupBy.count()` 返回包含运行计数的流`DataSet`。
* `foreach()` - 使用 `ds.writeStream.foreach（...）`（参见下一节）。
* `show()` - 而是使用控制台接收器

如果您尝试任何这些操作，您将看到一个 `AnalysisException` 如“操作 XYZ 不支持与流 `DataFrames/DataSet`”。

## 事件时间上的窗口操作
事件时间是嵌入在数据本身的时间，对于许多应用程序，我们可能希望根据事件时间进行聚合操作，为此，Spark2.x 提供了基于滑动窗口的事件时间集合操作。基于分组的聚合操作和基于窗口的聚合操作是非常相似的，在分组聚合中，依据用户指定的分组列中的每个唯一值维护聚合值，在基于窗口的聚合的情况下，对于行的事件时间落入的每个窗口维持聚合值。

![][1]
```
import spark.implicits._

val words = ... // streaming DataFrame of schema { timestamp: Timestamp, word: String }

// Group the data by window and word and compute the count of each group
val windowedCounts = words.groupBy(
  window($"timestamp", "10 minutes", "5 minutes"),
  $"word"
).count()
```
> 该段代码用于用于统计每10分钟内，接受到的不同词的个数，其中window($"timestamp", "10 minutes", "5 minutes")的含义为：假设初始时间 t=12:00，定义时间窗口为10分钟，每5分钟窗口滑动一次，也就是每5分钟对大小为10分钟的时间窗口进行一次聚合操作，并且聚合操作完成后，窗口向前滑动5分钟，产生新的窗口，如上图的一些列窗口 12:00-12:10,12:05-12:15,12:10-12:20。

*在这里每个word包含两个时间，word产生的时间和流接收到word的时间，这里的timestamp就是word产生的时间，在很多情况下，word产生后，可能会延迟很久才被流接收，为了处理这种情况，Structured Streaming 引进了Watermarking(时间水印)功能，以保证能正确的对流的聚合结构进行更新*

![][2]
Watermarking的计算方法[Watermarking](https://docs.google.com/document/d/1z-Pazs5v4rA31azvmYhu4I5xwqaNQl6ZLIS03xhkfCQ/edit#)：
- In every trigger, while aggregate the data, we also scan for the max value of event time in the trigger data
- After trigger completes, compute watermark = MAX(event time before trigger, max event time in trigger)

**Watermarking表示多长时间以前的数据将不再更新，也就是说每次窗口滑动之前会进行Watermarking的计算，首先统计这次聚合操作返回的最大事件时间，然后减去所然忍受的延迟时间就是Watermarking，当一组数据或新接收的数据事件时间小于Watermarking时，则该数据不会更新，在内存中就不会维护该组数据的状态**

![enter description here][3]
## Structured Streaming 支持两种更新模式：
1. `Update` 删除不再更新的时间窗口，每次触发聚合操作时，输出更新的窗口

![enter description here][4]
2. `Append` 当确定不会更新窗口时，将会输出该窗口的数据并删除，保证每个窗口的数据只会输出一次

![enter description here][5]
3. `Complete` 不删除任何数据，在 Result Table 中保留所有数据，每次触发操作输出所有窗口数据


  [1]: ./images/structured-streaming-window.png "structured-streaming-window"
  [2]: ./images/structured-streaming-late-data.png "structured-streaming-late-data"
  [3]: ./images/mw1.png "mw1"
  [4]: ./images/structured-streaming-watermark-update-mode.png "structured-streaming-watermark-update-mode"
  [5]: ./images/structured-streaming-watermark-append-mode.png "structured-streaming-watermark-append-mode"