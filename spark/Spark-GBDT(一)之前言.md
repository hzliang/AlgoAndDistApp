
# Spark GradientBoostedTrees(一)之前言

通过分析 `Spark` 机器学习源码，会发现有两种包结构：
```
org.apache.spark.mllib
org.apache.spark.ml
```
每种包中含有相应的机器学习算法实现，其中 `org.apache.spark.mllib` 是旧版的机器学习包入口，针对的输入是 `RDD`,而 `org.apache.spark.ml` 是新版的机器学习包入口，针对的输入是 `DataSet`。
以今天的主角 `GradientBoostedTrees` 为例，它在新旧版的包结构分别为:
```
org.apache.spark.mllib.tree.GradientBoostedTrees
org.apache.spark.ml.tree.impl.GradientBoostedTrees
```
接下来我们把 `GradientBoostedTrees` 简称为 `GBDT`，以下一系列文章将会介绍 `Spark GBDT` 的实现过程

## `GBDT` 的调用
```
  def train(
      input: RDD[LabeledPoint],
      boostingStrategy: BoostingStrategy): GradientBoostedTreesModel = {
    new GradientBoostedTrees(boostingStrategy, seed = 0).run(input)
  }
```
`GBDT` 的 `train` 方法新建一个 `GradientBoostedTrees` 对象，然后调用该对象的 `run` 方法：
```
  def run(input: RDD[LabeledPoint]): GradientBoostedTreesModel = {
    val algo = boostingStrategy.treeStrategy.algo
    val (trees, treeWeights) = NewGBT.run(input.map { point =>
      NewLabeledPoint(point.label, point.features.asML)
    }, boostingStrategy, seed.toLong)
    new GradientBoostedTreesModel(algo, trees.map(_.toOld), treeWeights)
  }
```
在 `run` 方法中，调用 `NewGBT` 的 `run` 方法，那么 `NewGBT` 是啥呢？其实它就是位于 `ml` 包中的 `GBDT` 算法实现，具体如 `import org.apache.spark.ml.tree.impl.{GradientBoostedTrees => NewGBT}`，所以` mllib` 中的 `GBDT` 最终调用的还是 `ml` 中的 `GBDT` 算法，这里主要做的工作就是把输入的 `LabeledPoint` 转成 `NewLabeledPoint`，如下 `org.apache.spark.ml.feature.{LabeledPoint => NewLabeledPoint}`，给人一种“挂羊头，卖狗肉”的感觉，有么有。
