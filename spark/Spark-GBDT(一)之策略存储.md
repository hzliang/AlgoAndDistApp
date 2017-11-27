
# Spark-GradientBoostedTrees前言之BoostingStrategy

`BoostingStrategy` 是 `GBDT` 算法的参数封装类，其构造函数如下：
```
case class BoostingStrategy @Since("1.4.0") (
    // Required boosting parameters
    @Since("1.2.0") @BeanProperty var treeStrategy: Strategy,
    @Since("1.2.0") @BeanProperty var loss: Loss,
    // Optional boosting parameters
    @Since("1.2.0") @BeanProperty var numIterations: Int = 100,
    @Since("1.2.0") @BeanProperty var learningRate: Double = 0.1,
    @Since("1.4.0") @BeanProperty var validationTol: Double = 0.001) extends Serializable
```
1. `numIterations` ：表示算法需要迭代多少次，每次迭代建立一颗回归树树🌲
2. `learningRate` ：`shrinking` 策略，指每次只学习一部分残差，而不是全部
3. `validationTol` ：迭代终止的一个验证条件
4. `loss` ：计算残差类
5. `treeStrategy` ：单颗回归树建立参数封装

`loss` 是 `Loss` 类型，表示损失函数，定义一个 `Loss` 类需要实现 `Loss` 的三个方法，如下：
```
trait Loss extends Serializable {
  def gradient(prediction: Double, label: Double): Double
  def computeError(model: TreeEnsembleModel, data: RDD[LabeledPoint]): Double = {
    data.map(point => computeError(model.predict(point.features), point.label)).mean()
  }
  private[spark] def computeError(prediction: Double, label: Double): Double
}
```
对于分类问题，需要继承自 `ClassificationLoss` ，并实现 `computeProbability` 方法，目前有三种代价函数，分别是：
1. `AbsoluteError` ：绝对值损失
2. `LogLoss` ：对数损失
3. `SquaredError` ：平方损失

其中 `AbsoluteError`，`SquaredError` 用于回归任务，`LogLoss` 用于分类任务。

`treeStrategy` 是 `Strategy` 类型，表示单颗回归树建立参数，定义如下：
```
    @Since("1.0.0") @BeanProperty var algo: Algo,
    @Since("1.0.0") @BeanProperty var impurity: Impurity,
    @Since("1.0.0") @BeanProperty var maxDepth: Int,
    @Since("1.2.0") @BeanProperty var numClasses: Int = 2,
    @Since("1.0.0") @BeanProperty var maxBins: Int = 32,
    @Since("1.0.0") @BeanProperty var quantileCalculationStrategy: QuantileStrategy = Sort,
    @Since("1.0.0") @BeanProperty var categoricalFeaturesInfo: Map[Int, Int] = Map[Int, Int](),
    @Since("1.2.0") @BeanProperty var minInstancesPerNode: Int = 1,
    @Since("1.2.0") @BeanProperty var minInfoGain: Double = 0.0,
    @Since("1.0.0") @BeanProperty var maxMemoryInMB: Int = 256,
    @Since("1.2.0") @BeanProperty var subsamplingRate: Double = 1,
    @Since("1.2.0") @BeanProperty var useNodeIdCache: Boolean = false,
    @Since("1.2.0") @BeanProperty var checkpointInterval: Int = 10) extends Serializable {
  @Since("1.2.0")
  def defaultStrategy(algo: String): Strategy = {
    defaultStrategy(Algo.fromString(algo))
  }

/**
 * Construct a default set of parameters for [[org.apache.spark.mllib.tree.DecisionTree]]
 * @param algo Algo.Classification or Algo.Regression
 */
@Since("1.3.0")
def defaultStrategy(algo: Algo): Strategy = algo match {
  case Algo.Classification =>
    new Strategy(algo = Classification, impurity = Gini, maxDepth = 10,
      numClasses = 2)
  case Algo.Regression =>
    new Strategy(algo = Regression, impurity = Variance, maxDepth = 10,
      numClasses = 0)
}
  }
```
1. `maxPossibleBins` ：`min(maxBins,样本连续特征数)`且必须大于 `categoricalFeaturesInfo` 中的最大的离散特征值数，`categoricalFeaturesInfo` 存储离散`特征index`，`特征值数`
2. `numBins` ：所有`特征index`及`其特征值数`，`Int`数组，维数是特征数，默认大小是 `maxPossibleBins`。对于连续特征，其值就是默认值 `maxPossibleBins`。对于离散特征，如为二分类或回归，此处将 `categoricalFeaturesInfo` 中的 `key`特征 `index` 作为数组 `index`，`value`特征个数写入数组中；如果是多分类，先计算其当作`UnorderedFeature`（无序的离散特征）的`bin`，如果个数小于等于`maxPossibleBins`，会被当成`UnorderedFeature`，否则被当成`orderedFeatures`（为了防止计算指数溢出，实际是把`maxPossibleBins`取`log`与特征数比较），因为`UnorderedFeature`的`bin`是比较大，这里限制了其特征值不能太多，这里仅仅根据特征值的特殊决定是否是`ordered`，不太好。每个`split`要将所有特征值分成两部分，`bin`的数量也就是`2*split`，因此`bin`的个数是`2*(2^(M-1)-1)`
3. `numFeaturesPerNode`：由`featureSubsetStrategy`决定，如果为`“auto”`，且为单棵树，则使用全部特征；如为多棵树，分类则是`sqrt`，回归为`1/3`；也可以自己指定，支持`”all”`, `“sqrt”`, `“log2”`, `“onethird”`，`GBDT`特征为`all`
4. `Strategy` 类的大部分参数有默认值，其中有个重要构建`Strategy` 对象的方法 `defaultStrategy`，用于指定任务类型、树节点不纯的评价函数以及类数。目前对于分类默认使用`Gini(基尼指数)`，对于回归默认使用`Variance(方差)`评价不纯，也可以自己指定评价函数，例如对`RandomForest`可以指定`Entropy(熵)`作为评价函数。

**注意：`GBDT`，`RandomForest`构造单颗树的逻辑复用的都是决策树的够着过程，只是参数不一样，并且`GBDT`支持连续和分类参数，如果样本中有分类参数，需要在`categoricalFeaturesInfo`中指定分类参数的特征`index`及特征值数。**
