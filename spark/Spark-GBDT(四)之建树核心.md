# Spark GradientBoostedTrees(四)建树核心
上一节介绍了`GBDT`建树的主要过程，这一节将介绍`selectNodesToSplit`和`findBestSplits`两个核心方法。
## `selectNodesToSplit`
1. `selectNodesToSplit`统计本次迭代应分裂的树节点，返回`(nodesForGroup, treeToNodeToIndexInfo)`二元组，其中`nodesForGroup`存储结构为`Map[Int, Array[LearningNode]]`，`key`为树的索引`treeIndex`，从`0`开始编号，`value`为`LearningNode`数组，即`treeIndex`对应的分裂节点数组，`treeToNodeToIndexInfo`存储结构为`Map[Int, Map[Int, NodeIndexInfo]]`，`key`为树的索引`treeIndex`，`value`为`Map`结构，`Map`的`key`为节点在树中被分配的`node.id`，每棵树的根节点编号为`1`，左子树节点编号`i*2`，右子树节点编号`i*2+1`，`NodeIndexInfo`是`numNodesInGroup`和`featureSubset`的包装对象，`numNodesInGroup`也是从`0`开始编号，对本轮要分裂的节点从`0~m`重新编号。
2. 方法内部主要通过迭代遍历`nodeStack`对象完成，`nodeStack`是上一轮分裂完成后，加入的需要分裂的新的节点，因为需要在每个树节点上存储按树路径分类后，属于该节点的样本数量，所有每个节点必须准备一定的缓存，默认每次分裂最大缓存为`val maxMemoryUsage: Long = strategy.maxMemoryInMB * 1024L * 1024L`，分类任务每个节点需要的缓存为`metadata.numClasses * totalBins`，回归任务需要存储`节点总数`，`正负样本之和`，`正负样本平方和`，每项都使用`double(8bit)`类型存储存储，最后消耗的总内存需要乘以`8`。
3. 统计`(nodesForGroup, treeToNodeToIndexInfo)`二元组代码如下
	```java
	  while (nodeStack.nonEmpty && (memUsage < maxMemoryUsage || memUsage == 0)) {
			val (treeIndex, node) = nodeStack.top
			// Choose subset of features for node (if subsampling).
			val featureSubset: Option[Array[Int]] = if (metadata.subsamplingFeatures) {
			  Some(SamplingUtils.reservoirSampleAndCount(Range(0,
				metadata.numFeatures).iterator, metadata.numFeaturesPerNode, rng.nextLong())._1)
			} else {
			  None
			}
			// Check if enough memory remains to add this node to the group.
			val nodeMemUsage = RandomForest.aggregateSizeForNode(metadata, featureSubset) * 8L
			if (memUsage + nodeMemUsage <= maxMemoryUsage || memUsage == 0) {
			  nodeStack.pop()
			  mutableNodesForGroup.getOrElseUpdate(treeIndex, new mutable.ArrayBuffer[LearningNode]()) +=
				node
			  mutableTreeToNodeToIndexInfo
				.getOrElseUpdate(treeIndex, new mutable.HashMap[Int, NodeIndexInfo]())(node.id)
				= new NodeIndexInfo(numNodesInGroup, featureSubset)
			}
			numNodesInGroup += 1
			memUsage += nodeMemUsage
		  }
		  if (memUsage > maxMemoryUsage) {
			// If maxMemoryUsage is 0, we should still allow splitting 1 node.
			logWarning(s"Tree learning is using approximately $memUsage bytes per iteration, which" +
			  s" exceeds requested limit maxMemoryUsage=$maxMemoryUsage. This allows splitting" +
			  s" $numNodesInGroup nodes in this iteration.")
		  }
		  // Convert mutable maps to immutable ones.
		  val nodesForGroup: Map[Int, Array[LearningNode]] =
			mutableNodesForGroup.mapValues(_.toArray).toMap
		  val treeToNodeToIndexInfo = mutableTreeToNodeToIndexInfo.mapValues(_.toMap).toMap
		  (nodesForGroup, treeToNodeToIndexInfo)
	```

## `findBestSplits`

1. `findBestSplits`首先创建一个`nodes`数组，存储本轮分裂中的`node`，数组索引使用`numNodesInGroup`，以便通过`group`获得最佳的分裂树节点。

	```
	  input.mapPartitions { points =>
		// Construct a nodeStatsAggregators array to hold node aggregate stats,
		// each node will have a nodeStatsAggregator
		val nodeStatsAggregators = Array.tabulate(numNodes) { nodeIndex =>
		  val featuresForNode = nodeToFeaturesBc.value.flatMap { nodeToFeatures =>
			Some(nodeToFeatures(nodeIndex))
		  }
		  new DTStatsAggregator(metadata, featuresForNode)
		}

		// iterator all instances in current partition and update aggregate stats
		points.foreach(binSeqOp(nodeStatsAggregators, _))

		// transform nodeStatsAggregators array to (nodeIndex, nodeAggregateStats) pairs,
		// which can be combined with other partition using `reduceByKey`
		nodeStatsAggregators.view.zipWithIndex.map(_.swap).iterator
	  }
	```
	> 对每个样本点，调用`binSeqOp`方法，该方法使用`predictImpl`函数判断当前样本点将在已构建的部分树的那个叶子节点上进行预测，从而获知该样本点会在哪个树节点上进行统计量计算，然后调用`nodeBinSeqOp`将统计量更新到`DTStatsAggregator`中，其中连续特征调用`orderedBinSeqOp`，含有分类特征则调用`mixedBinSeqOp`方法，内部主要是更新`DTStatsAggregator`在每个分裂点左右子树的样本统计量。
	> `val binIndex = treePoint.binnedFeatures(featureIndex)`
	> `agg.update(featureIndex, binIndex, label, instanceWeight)`

	```
	  def binSeqOp(
		  agg: Array[DTStatsAggregator],
		  baggedPoint: BaggedPoint[TreePoint]): Array[DTStatsAggregator] = {
		treeToNodeToIndexInfo.foreach { case (treeIndex, nodeIndexToInfo) =>
		  val nodeIndex =
			topNodesForGroup(treeIndex).predictImpl(baggedPoint.datum.binnedFeatures, splits)
		  nodeBinSeqOp(treeIndex, nodeIndexToInfo.getOrElse(nodeIndex, null), agg, baggedPoint)
		}
		agg
	  }
	```
3. 将分区数据合并，并计算最佳分裂点，内部通过`binsToBestSplit`方法得到最佳分裂点，由于`DTStatsAggregator`只是记录了每一个样本的最佳分裂位置`(二分搜索)`的统计量，为了获得某个分裂点左右子树所有的样本统计量，`binsToBestSplit`对`DTStatsAggregator`从左往右做累加`(merge)`，这样总的累加和减掉分裂点左边的累加和就等于分裂点右边的累加和，最有对左右两侧的节点计算不纯度，增益最大的分裂点作为最佳分裂点。

    ```
    val nodeToBestSplits = partitionAggregates.reduceByKey((a, b) => a.merge(b)).map {
      case (nodeIndex, aggStats) =>
        val featuresForNode = nodeToFeaturesBc.value.flatMap { nodeToFeatures =>
          Some(nodeToFeatures(nodeIndex))
        }
        // find best split for each node
        val (split: Split, stats: ImpurityStats) =
          binsToBestSplit(aggStats, splits, featuresForNode, nodes(nodeIndex))
        (nodeIndex, (split, stats))
    }.collectAsMap()


4. `binsToBestSplit`方法首先获得要分裂节点的层，以识别是否是树根节点，根节点返回`null`，根节点最初没有统计量，需要通过左右子节点统计量做`merge`获得。

    ```
	  val level = LearningNode.indexToLevel(node.id)
	  var gainAndImpurityStats: ImpurityStats = if (level == 0) {
		null
	  } else {
		node.stats
	  }
    ```

5. 对`DTStatsAggregator`从左往右做累加，然后对累加后的分割点一个个判断，选择增益最大的节点作为最佳的分裂节点，增益的计算通过calculateImpurityStats方法

	```
	  val nodeFeatureOffset = binAggregates.getFeatureOffset(featureIndexIdx)
	  var splitIndex = 0
	  while (splitIndex < numSplits) {
	   binAggregates.mergeForFeature(nodeFeatureOffset, splitIndex + 1, splitIndex)
	   splitIndex += 1
	  }
	  // Find best split.
	  val (bestFeatureSplitIndex, bestFeatureGainStats) =
	   Range(0, numSplits).map { case splitIdx =>
		 val leftChildStats = binAggregates.getImpurityCalculator(nodeFeatureOffset, splitIdx)
		 val rightChildStats =
		   binAggregates.getImpurityCalculator(nodeFeatureOffset, numSplits)
		 rightChildStats.subtract(leftChildStats)
		 gainAndImpurityStats = calculateImpurityStats(gainAndImpurityStats,
		   leftChildStats, rightChildStats, binAggregates.metadata)
		 (splitIdx, gainAndImpurityStats)
	   }.maxBy(_._2.gain)
	  (splits(featureIndex)(bestFeatureSplitIndex), bestFeatureGainStats)   
	```

6. 信息增益

	```
	  val leftCount = leftImpurityCalculator.count
	  val rightCount = rightImpurityCalculator.count

	  val totalCount = leftCount + rightCount

	  // If left child or right child doesn't satisfy minimum instances per node,
	  // then this split is invalid, return invalid information gain stats.
	  if ((leftCount < metadata.minInstancesPerNode) ||
		(rightCount < metadata.minInstancesPerNode)) {
		return ImpurityStats.getInvalidImpurityStats(parentImpurityCalculator)
	  }

	  val leftImpurity = leftImpurityCalculator.calculate() // Note: This equals 0 if count = 0
	  val rightImpurity = rightImpurityCalculator.calculate()

	  val leftWeight = leftCount / totalCount.toDouble
	  val rightWeight = rightCount / totalCount.toDouble

	  val gain = impurity - leftWeight * leftImpurity - rightWeight * rightImpurity
	```
