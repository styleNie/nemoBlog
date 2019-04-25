<!-- 
---
title: "GBDT和XGBoost介绍（四）"    
author:     
date: February 18, 2019    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
---
-->
# <center>GBDT和XGBoost介绍（四）</center>  
 

>转自https://zhuanlan.zhihu.com/p/26154587

本文是GBDT算法的第四篇，在完成XGBoost的基本介绍与数学推导后，接下来学习XGBoost区别于GBDT的一些独特之处以及算法的R实现。    

## 一、XGBoost的优良特性
同样是梯度提升，同样是集成学习，那么XGBoost比GBDT要好在哪里呢？结合前面的推导过程与相关博客文章（见文末参考资料），可大致总结为以下几点：

- GBDT是以CART为基分类器，但XGBoost在此基础上还支持线性分类器，此时XGBoost相当于带$L_1$和$L_2$正则化项的Logistics回归（分类问题）或者线性回归（回归问题）
- XGBoost在目标函数里加入了正则项，用于控制模型的复杂度。正则项里包含了树的叶子节点个数和每棵树叶子节点上面输出分数的$L_2$模平方。从偏差方差权衡的角度来讲，正则项降低了模型的variance，使学习出来的模型更加简单，防止过拟合
- 传统的GBDT在优化时只用到一阶导数，XGBoost则对目标函数进行了二阶泰勒展开，同时用到了一阶和二阶导数。（顺便提一下，XGBoost工具支持自定义代价函数，只要函数可一阶和二阶求导）
- 树节点在进行分裂时，我们需要计算每个特征的每个分割点对应的增益，即用贪心法枚举所有可能的分割点。当数据无法一次载入内存或者在分布式情况下，贪心算法效率就会变得很低，所以XGBoost采用了一种近似的算法。大致的思想是根据百分位法列举几个可能成为分割点的候选者，然后从候选者中根据上面求分割点的公式计算找出最佳的分割点
- Shrinkage（缩减），相当于学习速率（XGBoost中的eta）。XGBoost在进行完一次迭代后，会将叶子节点的权重乘上该系数，主要是为了削弱每棵树的影响，让后面有更大的学习空间。实际应用中，一般把eta设置得小一点，然后迭代次数设置得大一点。（当然普通的GBDT实现也有学习速率）
- 特征列排序后以块的形式存储在内存中，在迭代中可以重复使用；虽然boosting算法迭代必须串行，但是在处理每个特征列时可以做到并行
- 列抽样（column subsampling）：XGBoost借鉴了随机森林的做法，支持列抽样，不仅能降低过拟合，还能减少计算，这也是XGBoost异于传统GBDT的一个特性
- 除此之外，XGBoost还考虑了当数据量比较大，内存不够时怎么有效的使用磁盘，主要是结合多线程、数据压缩、分片的方法，尽可能的提高算法效率    


## 二、xgboost包安装与数据准备  
在R中，xgboost包用于算法的实现，首先进行安装
```r
# xgboost包在安装时需要把R升级到3.3.0以上的版本,否则安装不成功
> install.packages('xgboost')
# 也可使用devtools包安装github版本
> devtools::install_github('dmlc/xgboost', subdir='R-package')
> library(xgboost)
```    
在包中有一组蘑菇数据集可供使用，我们的目标是预测蘑菇是否可以食用（分类任务），此数据集已被分割成训练数据与测试数据。
```r
> data(agaricus.train, package='xgboost')
> data(agaricus.test, package='xgboost')
> train <- agaricus.train 
> test <- agaricus.test
```

```r
# 整个数据集是由data和label组成的list
> class(train)
[1] "list"
# 查看数据维度
> dim(train$data)
[1] 6513  126
> dim(test$data)
[1] 1611  126
```

```r
# 在此数据集中，data是一个dgCMatrix类的稀疏矩阵,label是一个由{0,1}构成的数值型向量
> str(train)
List of 2
 $ data :Formal class 'dgCMatrix' [package "Matrix"] with 6 slots
  .. ..@ i       : int [1:143286] 2 6 8 11 18 20 21 24 28 32 ...
  .. ..@ p       : int [1:127] 0 369 372 3306 5845 6489 6513 8380 8384 10991 ...
  .. ..@ Dim     : int [1:2] 6513 126
  .. ..@ Dimnames:List of 2
  .. .. ..$ : NULL
  .. .. ..$ : chr [1:126] "cap-shape=bell" "cap-shape=conical" "cap-shape=convex" "cap-shape=flat" ...
  .. ..@ x       : num [1:143286] 1 1 1 1 1 1 1 1 1 1 ...
  .. ..@ factors : list()
 $label: num [1:6513] 1 0 0 1 0 0 0 1 0 0 ...
```     


## 三、构建模型和预测实现      
xgboost包提供了两个函数用于模型构建，分别是xgboost()与xgb.train()，前者可以满足对算法参数的基本设置，而后者的话在此基础上可以实现一些更为高级的功能。    
```r
# data与label分别指定数据与标签   
# max.deph：树的深度,默认值为6,在此数据集中的分类问题比较简单，设置为2即可   
# nthread：并行运算的CPU的线程数,设置为2;   
# nround：生成树的棵数   
# objective = "binary:logistic"：设置逻辑回归二分类模型   
> xgboost_model <- xgboost(data = train$data, label = train$label, max.depth = 2, eta = 1, nthread = 2, nround = 2, objective = "binary:logistic")   
# 得到两次迭代的训练误差   
[1]	train-error:0.046522
[2]	train-error:0.022263
```      

xgboost函数可调用的参数众多，在此不在详细展开介绍，可参阅博客文章[[译]快速上手：在R中使用XGBoost算法](https://segmentfault.com/a/1190000004421821#articleHeader7)中的"在xgboost中使用参数"一节，该文章将这些参数归为通用、辅助和任务参数三大类，对我们掌握算法与调参有着很大帮助。    

```r
# 设置verbose参数,可以显示内部的学习过程   
> xgboost_model <- xgboost(data = train$data, label = train$label, 
+ max.depth = 2, eta = 1, nthread = 2, nround = 2, verbose = 2,
+ objective = "binary:logistic")
[13:56:36] amalgamation/../src/tree/updater_prune.cc:74: tree pruning end, 1 roots, 6 extra nodes, 0 pruned nodes, max_depth=2
[1]	train-error:0.046522 
[13:56:36] amalgamation/../src/tree/updater_prune.cc:74: tree pruning end, 1 roots, 4 extra nodes, 0 pruned nodes, max_depth=2
[2]	train-error:0.022263
```    


```r
# 将建立好的模型用于预测新的数据集
> xgboost_pred <- predict(xgboost_model, test$data)
> head(xgboost_pred)
[1] 0.28583017 0.92392391 0.28583017 0.28583017 0.05169873 0.92392391
# 以上给出的是每一个样本的预测概率值,进一步转化后可得到具体的预测分类
> prediction <- as.numeric(xgboost_pred > 0.5)
> head(prediction)
[1] 0 1 0 0 0 1
> model_accuracy <- table(prediction,test$label)
> model_accuracy
          
prediction   0   1
         0 813  13
         1  22 763
> model_accuracy_1 <- sum(diag(model_accuracy))/sum(model_accuracy)
> model_accuracy_1
[1] 0.9782744
```

## 四、XGBoost的高级功能
xgb.train()函数可以实现一些高级功能，帮助我们对模型进行进一步的优化。
```r
# 在使用函数前需要将数据集进行转换为xgb.Dmatrix格式   
> dtrain <- xgb.DMatrix(data = train$data, label=train$label)   
> dtest <- xgb.DMatrix(data = test$data, label=test$label)   
```   

```r
# 使用watchlist参数,可同时得到训练数据与测试数据的误差
> watchlist <- list(train=dtrain, test=dtest)
> xgboost_model <- xgb.train(data=dtrain, max.depth=2, eta=1, nthread = 2,
+ nround = 3,objective = "binary:logistic",watchlist = watchlist)
[1]	train-error:0.046522	test-error:0.042831 
[2]	train-error:0.022263	test-error:0.021726
[3]	train-error:0.007063	test-error:0.006207
```

```r
# 自定义损失函数,可同时观察两种损失函数的表现
# eval.metric可使用的参数包括'logloss'、'error'、'rmse'等
> xgboost_model <- xgb.train(data=dtrain, max.depth=2, eta=1, nthread = 2,
+ nround=3, watchlist=watchlist, eval.metric = "error", 
+ eval.metric = "logloss", objective = "binary:logistic")
[1]	train-error:0.046522	train-logloss:0.233376	test-error:0.042831	test-logloss:0.226686 
[2]	train-error:0.022263	train-logloss:0.136658	test-error:0.021726	test-logloss:0.137874 
[3]	train-error:0.007063	train-logloss:0.082531	test-error:0.006207	test-logloss:0.080461   
```   

```r
# 查看特征的重要性,方便我们在模型优化时进行特征筛选
> importance_matrix <- xgb.importance(model = xgboost_model)
> importance_matrix
   Feature       Gain      Cover Frequency
1:      28 0.60036585 0.41841659     0.250
2:      55 0.15214681 0.16140352     0.125
3:      59 0.10936624 0.13772146     0.125
4:     101 0.04843973 0.07979724     0.125
5:     110 0.03391602 0.04120512     0.125
6:      66 0.02973248 0.03859211     0.125
7:     108 0.02603288 0.12286396     0.125
# 使用xgb.plot.importance()函数进行可视化展示
> xgb.plot.importance(importance_matrix)
```   


```r
# 使用xgb.dump()查看模型的树结构
> xgb.dump(xgboost_model,with_stats = T)
 [1] "booster[0]"                                                            
 [2] "0:[f28<-9.53674e-007] yes=1,no=2,missing=1,gain=4000.53,cover=1628.25" 
 [3] "1:[f55<-9.53674e-007] yes=3,no=4,missing=3,gain=1158.21,cover=924.5"   
 [4] "3:leaf=0.513653,cover=812"                                             
 [5] "4:leaf=-0.510132,cover=112.5"                                          
 [6] "2:[f108<-9.53674e-007] yes=5,no=6,missing=5,gain=198.174,cover=703.75" 
 [7] "5:leaf=-0.582213,cover=690.5"                                          
 [8] "6:leaf=0.557895,cover=13.25"   
 ---
# 将上述结果通过树形结构图表达出来    
> xgb.plot.tree(model = xgboost_model)    
```      
<img src=./pictures/gbdt_12.jpg>   

至此，XGBoost算法及其R实现就简单介绍到这里。虽然貌似讲了好多，但我们所学的不过是一些皮毛而已，无论是XGBoost本身所具有的优良性能、通过复杂的调参对不同任务的实现支持，还是在实际应用中的高精度预测，这些优势都将意味着XGBoost算法有着巨大的潜力空间，值得我们一直探索下去。    


## References：

1. [Get Started with XGBoost](https://xgboost.readthedocs.io/en/latest/get_started/index.html#r)
2. [严酷的魔王：xgboost：速度快效果好的boosting模型 | 统计之都](https://cosx.org/2015/03/xgboost)
3. [xgboost：速度快效果好的Boosting模型](https://mp.weixin.qq.com/s?__biz=MzI1NDMyMjgyMA==&mid=2247483764&idx=1&sn=2cbaf871c9f4579cefe34b3805a22557&mpshare=1&scene=24&srcid=0326t6677DyczoesfFZwX9kl#rd)
4. [机器学习算法中GBDT和XGBOOST的区别有哪些？ - 知乎](https://www.zhihu.com/question/41354392)
5. [[译]快速上手：在R中使用XGBoost算法 - FinanceR](https://segmentfault.com/a/1190000004421821#articleHeader9)