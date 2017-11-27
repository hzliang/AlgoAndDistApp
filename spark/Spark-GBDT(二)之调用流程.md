# Spark GradientBoostedTrees(二)调用流程
上节我们知道 `mllib` 中的 `GBDT` 算法最终会调用`ml` 中的 `GBDT` 算法，这节主要分析 `ml` 的 `GBDT` 算法调用过程：
```
    val algo = boostingStrategy.treeStrategy.algo
    algo match {
      case OldAlgo.Regression =>
        GradientBoostedTrees.boost(input, input, boostingStrategy, validate = false, seed)
      case OldAlgo.Classification =>
        // Map labels to -1, +1 so binary classification can be treated as regression.
        val remappedInput = input.map(x => new LabeledPoint((x.label * 2) - 1, x.features))
        GradientBoostedTrees.boost(remappedInput, remappedInput, boostingStrategy, validate = false,
          seed)
      case _ =>
        throw new IllegalArgumentException(s"$algo is not supported by gradient boosting.")
    }
```
`GBDT` 算法针对不同的任务(分类、回归)进行相应的处理，对于 `Regression`(回归分析），直接调用算法的`boost` 方法，而对于 `Classification`(分类分析)，则将 `0，1` 类标签转成 `-1，1` 表示，调用 `boost` 方法。
```
def boost(
      input: RDD[LabeledPoint],
      validationInput: RDD[LabeledPoint],
      boostingStrategy: OldBoostingStrategy,
      validate: Boolean,
      seed: Long): (Array[DecisionTreeRegressionModel], Array[Double]) = {
    val timer = new TimeTracker()
    timer.start("total")
    timer.start("init")

    boostingStrategy.assertValid()

    // Initialize gradient boosting parameters
    val numIterations = boostingStrategy.numIterations
    val baseLearners = new Array[DecisionTreeRegressionModel](numIterations)
    val baseLearnerWeights = new Array[Double](numIterations)
    val loss = boostingStrategy.loss
    val learningRate = boostingStrategy.learningRate
    // Prepare strategy for individual trees, which use regression with variance impurity.
    val treeStrategy = boostingStrategy.treeStrategy.copy
    val validationTol = boostingStrategy.validationTol
    treeStrategy.algo = OldAlgo.Regression
    treeStrategy.impurity = OldVariance
    treeStrategy.assertValid()

    // Cache input
    val persistedInput = if (input.getStorageLevel == StorageLevel.NONE) {
      input.persist(StorageLevel.MEMORY_AND_DISK)
      true
    } else {
      false
    }

    // Prepare periodic checkpointers
    val predErrorCheckpointer = new PeriodicRDDCheckpointer[(Double, Double)](
      treeStrategy.getCheckpointInterval, input.sparkContext)
    val validatePredErrorCheckpointer = new PeriodicRDDCheckpointer[(Double, Double)](
      treeStrategy.getCheckpointInterval, input.sparkContext)

    timer.stop("init")

    logDebug("##########")
    logDebug("Building tree 0")
    logDebug("##########")

    // Initialize tree
    timer.start("building tree 0")
    val firstTree = new DecisionTreeRegressor().setSeed(seed)
    val firstTreeModel = firstTree.train(input, treeStrategy)
    val firstTreeWeight = 1.0
    baseLearners(0) = firstTreeModel
    baseLearnerWeights(0) = firstTreeWeight

    var predError: RDD[(Double, Double)] =
      computeInitialPredictionAndError(input, firstTreeWeight, firstTreeModel, loss)
    predErrorCheckpointer.update(predError)
    logDebug("error of gbt = " + predError.values.mean())

    // Note: A model of type regression is used since we require raw prediction
    timer.stop("building tree 0")

    var validatePredError: RDD[(Double, Double)] =
      computeInitialPredictionAndError(validationInput, firstTreeWeight, firstTreeModel, loss)
    if (validate) validatePredErrorCheckpointer.update(validatePredError)
    var bestValidateError = if (validate) validatePredError.values.mean() else 0.0
    var bestM = 1

    var m = 1
    var doneLearning = false
    while (m < numIterations && !doneLearning) {
      // Update data with pseudo-residuals
      val data = predError.zip(input).map { case ((pred, _), point) =>
        LabeledPoint(-loss.gradient(pred, point.label), point.features)
      }

      timer.start(s"building tree $m")
      logDebug("###################################################")
      logDebug("Gradient boosting tree iteration " + m)
      logDebug("###################################################")
      val dt = new DecisionTreeRegressor().setSeed(seed + m)
      val model = dt.train(data, treeStrategy)
      timer.stop(s"building tree $m")
      // Update partial model
      baseLearners(m) = model
      // Note: The setting of baseLearnerWeights is incorrect for losses other than SquaredError.
      //       Technically, the weight should be optimized for the particular loss.
      //       However, the behavior should be reasonable, though not optimal.
      baseLearnerWeights(m) = learningRate

      predError = updatePredictionError(
        input, predError, baseLearnerWeights(m), baseLearners(m), loss)
      predErrorCheckpointer.update(predError)
      logDebug("error of gbt = " + predError.values.mean())

      if (validate) {
        // Stop training early if
        // 1. Reduction in error is less than the validationTol or
        // 2. If the error increases, that is if the model is overfit.
        // We want the model returned corresponding to the best validation error.

        validatePredError = updatePredictionError(
          validationInput, validatePredError, baseLearnerWeights(m), baseLearners(m), loss)
        validatePredErrorCheckpointer.update(validatePredError)
        val currentValidateError = validatePredError.values.mean()
        if (bestValidateError - currentValidateError < validationTol * Math.max(
          currentValidateError, 0.01)) {
          doneLearning = true
        } else if (currentValidateError < bestValidateError) {
          bestValidateError = currentValidateError
          bestM = m + 1
        }
      }
      m += 1
    }

    timer.stop("total")

    logInfo("Internal timing for DecisionTree:")
    logInfo(s"$timer")

    predErrorCheckpointer.deleteAllCheckpoints()
    validatePredErrorCheckpointer.deleteAllCheckpoints()
    if (persistedInput) input.unpersist()

    if (validate) {
      (baseLearners.slice(0, bestM), baseLearnerWeights.slice(0, bestM))
    } else {
      (baseLearners, baseLearnerWeights)
    }
  }
```
1. `boost` 方法首先对参数初始化，这里需要注意的是在初始化参数的过程中，会 `copy` 一个 `treeStrategy` 对象，原始的 `treeStrategy` 对象用于计算残差，新对象用于训练，新对象将
`algo` 参数设置为 `OldAlgo.Regression`，也就是建立回归树，将节点的不纯度定义为 `OldVariance`，然后新建 `DecisionTreeRegressor` 对象，开始训练第一颗回归树。
```
    val treeStrategy = boostingStrategy.treeStrategy.copy
    treeStrategy.algo = OldAlgo.Regression
    treeStrategy.impurity = OldVariance
``` 
2. 第一颗回归树构建完成后，将会计算得到的集成回归树对每个样本的预测，以得到残差(使用代价函数的负梯度作为伪残差)，将残差当作第二颗树的输入，继续训练，直到创建指定数量的回归树为止。


