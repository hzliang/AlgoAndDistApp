
# Spark GradientBoostedTrees(三)回归树建立
上一节我们分析了 `GBDT` 的调度流程，其中最关键的是如何建立一颗回归树，接下来我们将分析回归树的构建过程：
1. 构造元数据

	```
	val metadata =
		  DecisionTreeMetadata.buildMetadata(retaggedInput, strategy, numTrees, featureSubsetStrategy)
	```
	
	`buildMetadata` 方法内部实现逻辑：
	 * 首先计算特征总数`numFeatures` 和 `maxPossibleBins(所有特征的最大分割数)`，`maxPossibleBins`由我们指定的 `maxBins` 和 `样本总量` 决定
	`val maxPossibleBins = math.min(strategy.maxBins, numExamples).toInt`
	 * 然后会判断分类变量的`特征值数`，分类变量的特征值数必须小于 `maxPossibleBins`
	 * 最后计算每个特征的分割数`numBins`，对于分类变量，分割数等于特征值数
	 * 构建 `DecisionTreeMetadata` 对象
	```
		new DecisionTreeMetadata(numFeatures, numExamples, numClasses, numBins.max,
		  strategy.categoricalFeaturesInfo, unorderedFeatures.toSet, numBins,
		  strategy.impurity, strategy.quantileCalculationStrategy, strategy.maxDepth,
		  strategy.minInstancesPerNode, strategy.minInfoGain, numTrees, numFeaturesPerNode)
	```
* `numClasses`：类别数，构建回归树时为`0`
* `numFeaturesPerNode`：每个树节点的特征总数，由`featureSubsetStrategy`决定，主要用于`RandomForest`，`GBDT`返回所有特征

2. 计算每个特征的所有可选分割阈值

	```
	val splits = findSplits(retaggedInput, metadata, seed)
	```
	
`findSplits` 只计算连续特征的分割，对于分类特征，直接返回空数组，内部逻辑如下：
 * 先对数据采样，因为使用采用数据，算法能够得到更好的效果

	```
	val requiredSamples = math.max(metadata.maxBins * metadata.maxBins, 10000)

	```

 * 对于得到的采样数据调用 `findSplitsBySorting` 方法
 * 
	 ```
	 findSplitsBySorting(sampledInput, metadata, continuousFeatures)
	 ```
	 
`findSplitsBySorting` 方法对于分类特征直接返回空数组，对于每个连续特征，调用 `findSplitsForContinuousFeature` 方法获得所有可能的分割，将返回的每个分割点包装成`ContinuousSplit`对象，`findSplitsForContinuousFeature`方法内部逻辑如下：
 * 首先统计每个连续特征所有的值及出现的次数，`唯一特征的总数-1`即为分割点数
 * 然后按值从小打到大排序，如果分割点为`0`，返回空数组，如果分割点数小于等于`numSplits`，则每个分割点为前后两个特征值的均值，否则，按步长做分割(这里原理没看懂😔)

	```
	  def numSplits(featureIndex: Int): Int = if (isUnordered(featureIndex)) {
		numBins(featureIndex)
	  } else {
		numBins(featureIndex) - 1
	  }
	```

3. 将`LabeledPoint`对象转换成`TreePoint`，方法计算了`thresholds`与`featureArity`，最后调用方法`TreePoint.labeledPointToTreePoint`，对连续特征使用二分搜索计算每个样本数据点中的每个特征值，在该特征所有分割点的最佳分割阈值及索引，分类特征直接返回特征值，以备计算最佳分割点

	```
	  val treeInput = TreePoint.convertToTreeRDD(retaggedInput, splits, metadata)
	  def convertToTreeRDD(
		  input: RDD[LabeledPoint],
		  splits: Array[Array[Split]],
		  metadata: DecisionTreeMetadata): RDD[TreePoint] = {
		// Construct arrays for featureArity for efficiency in the inner loop.
		val featureArity: Array[Int] = new Array[Int](metadata.numFeatures)
		var featureIndex = 0
		while (featureIndex < metadata.numFeatures) {
		  featureArity(featureIndex) = metadata.featureArity.getOrElse(featureIndex, 0)
		  featureIndex += 1
		}
		val thresholds: Array[Array[Double]] = featureArity.zipWithIndex.map { case (arity, idx) =>
		  if (arity == 0) {
			splits(idx).map(_.asInstanceOf[ContinuousSplit].threshold)
		  } else {
			Array.empty[Double]
		  }
		}
		input.map { x =>
		  TreePoint.labeledPointToTreePoint(x, thresholds, featureArity)
		}
	  }
	```
	
4. 将`TreePoint`转换成`BaggedPoint`对象，内部为每个样本赋权重，对于`GBDT`算法，所有样本的权重设置为`1.0`

	```
		val baggedInput = BaggedPoint
		  .convertToBaggedRDD(treeInput, strategy.subsamplingRate, numTrees, withReplacement, seed)
		  .persist(StorageLevel.MEMORY_AND_DISK)
	```
	
5. 构造每颗树的根节点，对于`GBDT`，只能在单颗树上并行，所有每次只构建一颗回归树，`RandomForest`则并行构建多颗决策树

	```
		val nodeStack = new mutable.Stack[(Int, LearningNode)]

		val rng = new Random()
		rng.setSeed(seed)

		// Allocate and queue root nodes.
		val topNodes = Array.fill[LearningNode](numTrees)(LearningNode.emptyNode(nodeIndex = 1))
		Range(0, numTrees).foreach(treeIndex => nodeStack.push((treeIndex, topNodes(treeIndex))))
	```
	
6. 调用`selectNodesToSplit`方法返回该次需要分裂的所有节点，调用`findBestSplits`找到最佳的分裂点，这两个方法是树构造的核心，将在下一节详细介绍

	```
	while (nodeStack.nonEmpty) {
		  // Collect some nodes to split, and choose features for each node (if subsampling).
		  // Each group of nodes may come from one or multiple trees, and at multiple levels.
		  val (nodesForGroup, treeToNodeToIndexInfo) =
			RandomForest.selectNodesToSplit(nodeStack, maxMemoryUsage, metadata, rng)
		  // Sanity check (should never occur):
		  assert(nodesForGroup.nonEmpty,
			s"RandomForest selected empty nodesForGroup.  Error for unknown reason.")

		  // Only send trees to worker if they contain nodes being split this iteration.
		  val topNodesForGroup: Map[Int, LearningNode] =
			nodesForGroup.keys.map(treeIdx => treeIdx -> topNodes(treeIdx)).toMap

		  // Choose node splits, and enqueue new nodes as needed.
		  timer.start("findBestSplits")
		  RandomForest.findBestSplits(baggedInput, metadata, topNodesForGroup, nodesForGroup,
			treeToNodeToIndexInfo, splits, nodeStack, timer, nodeIdCache)
		  timer.stop("findBestSplits")
		}
	```
7. 将决策树信息包装成`DecisionTreeClassificationModel`对象，需要注意的是在构建对象的过程中，会调用`rootNode.toNode`方法将`LearnNode`转换成`Node`
* 非叶子节点

	```
		  new InternalNode(stats.impurityCalculator.predict, stats.impurity, stats.gain,
			leftChild.get.toNode, rightChild.get.toNode, split.get, stats.impurityCalculator)
	 ```
	 
* 叶子节点

	 ```
		   new LeafNode(stats.impurityCalculator.predict, stats.impurity,
			 stats.impurityCalculator)
	```

* 构造`DecisionTreeClassificationModel`对象

	```
		parentUID match {
		  case Some(uid) =>
			if (strategy.algo == OldAlgo.Classification) {
			  topNodes.map { rootNode =>
				new DecisionTreeClassificationModel(uid, rootNode.toNode, numFeatures,
				  strategy.getNumClasses)
			  }
			} else {
			  topNodes.map { rootNode =>
				new DecisionTreeRegressionModel(uid, rootNode.toNode, numFeatures)
			  }
			}
		  case None =>
			if (strategy.algo == OldAlgo.Classification) {
			  topNodes.map { rootNode =>
				new DecisionTreeClassificationModel(rootNode.toNode, numFeatures,
				  strategy.getNumClasses)
			  }
			} else {
			  topNodes.map(rootNode => new DecisionTreeRegressionModel(rootNode.toNode, numFeatures))
			}
		}
	```
