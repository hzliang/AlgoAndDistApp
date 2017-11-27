# Spark RecurringTimer
`RecurringTime` 类是 `Spark` 自定义的**定时执行类**，简单的看下它的实现过程：

构造函数：
```
class RecurringTimer(clock: Clock, period: Long, callback: (Long) 
    => Unit, name: String) extends Logging
```
`Clock` 是 `Spark` 自定义的时间类，
```
private[spark] trait Clock {
  def getTimeMillis(): Long
  def waitTillTime(targetTime: Long): Long
}
```
它的唯一实现类是 `SystemClock`，实现了 `waitTillTime(targetTime: Long)` 函数，用于阻塞线程直到 `targetTime` 为止

`peroid` 为定时器再次执行的间隔时间

`callback` 为定时器执行的函数

定时器通过自定义线程来实现：
```
  private val thread = new Thread("RecurringTimer - " + name) {
    setDaemon(true)
    override def run() { loop }
  }
```
`run` 方法调用 `loop` 方法，循环等待执行的到来时间：
```
  private def loop() {
    try {
      while (!stopped) {
        triggerActionForNextInterval()
      }
      triggerActionForNextInterval()
    } catch {
      case e: InterruptedException =>
    }
  }
```
`triggerActionForNextInterval` 方法也就是我们需要定时器定时执行的方法：
```
  private def triggerActionForNextInterval(): Unit = {
    clock.waitTillTime(nextTime)
    callback(nextTime)
    prevTime = nextTime
    nextTime += period
    logDebug("Callback for " + name + " called at time " + prevTime)
  }
```
当调用 `RecurringTimer` 的 `start` 方法时，并没有立刻执行定时器，而是计算什么时间定时器第一次执行，
```
	def start(): Long = {
	    start(getStartTime())
	}
	
	def getRestartTime(originalStartTime: Long): Long = {
        val gap = clock.getTimeMillis() - originalStartTime

	def start(startTime: Long): Long = synchronized {
	    nextTime = startTime
	    thread.start()
	    logInfo("Started timer for " + name + " at time " + nextTime)
	    nextTime
	}
```
这样做是为了使得每次执行时间为 `peroid` 的整数倍