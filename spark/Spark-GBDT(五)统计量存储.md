# Spark GBDT(五)节点统计量信息存储类
`DecisionTree`使用`DTStatsAggregator`对象包装每个节点统计量，`DTStatsAggregator`主要包括以下几个参数：

* `statsSize: Int`：每个特征需要存储多少个统计量，例如，回归任务每个特征会统计在每个分割点的三种信息，所以`statsSize=3`

	```
	allStats(offset) += instanceWeight
	allStats(offset + 1) += instanceWeight * label
	allStats(offset + 2) += instanceWeight * label * label
	```
* `numBins: Array[Int]`：每个特征的分割点数量
* `allStats: Array[Double] = new Array[Double](allStatsSize)`：存储统计量的数组
* `featureOffsets: Array[Int]`：每个特征存储的统计量在`allStats`中的偏移

	```
	numBins.scanLeft(0)((total, nBins) => total + statsSize * nBins)
	```

例：假设有特征`a，b，c`，分割数分别为`4，5，6`，计算回归任务，使用方差作为不纯度衡量，则`allStats`长度为`45`，`0~3`存储的是`a`的第一个特征第一个分割点的统计量，`4~6`存储的是`a`的第一个特征第二个分割点的统计量，依次类推。**注：每个样本在计算统计量时，使用二分搜索法(回归任务，每一位特征值从小到大排序)计算其最佳(最近)的分割点索引，按该索引更新`allStats`，所以为了得到分割点左右两侧的累积统计量，程序中做了从左到右的累加。**

* 最后还有一个核心对象`impurityAggregator `，用于计算树节点的不纯度，目前`Spark 2.1.1`提供三种不纯度计算对象，每一种对象都实现了`update`方法，用于自定义统计量的更新操作和`getCalculator`方法，用于获取封装`不纯度计算`、`预测分类`、`预测概率`的对象。

	```
	val impurityAggregator: ImpurityAggregator = metadata.impurity match {
	case Gini => new GiniAggregator(metadata.numClasses)
	case Entropy => new EntropyAggregator(metadata.numClasses)
	case Variance => new VarianceAggregator()
	case _ => throw new IllegalArgumentException(s"Bad impurity parameter: ${metadata.impurity}")
	}
	```
