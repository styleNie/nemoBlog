<!-- 
---
title: "GBDT和XGBoost介绍（一）"    
author:     
date: February 18, 2019    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
---
-->
# <center>GBDT和XGBoost介绍（一）</center>  


>转自https://zhuanlan.zhihu.com/p/25705586    
## 前言
GBDT（Gradient Boosting Decision Tree）是一种基于迭代所构造的决策树算法，它又可以简称为MART（Multiple Additive Regression Tree）或GBRT（Gradient Boosting Regression Tree）。虽然名字上又是Gradient又是Boosting的，但它的原理还是很浅显易懂（当然详细的推导还是有一些难度）。简单来讲，这种算法在实际问题中将生成多棵决策树，并将所有树的结果进行汇总来得到最终答案。也就是说，该算法将决策树与集成思想进行了有效的结合。

前面我们已经学过，集成思想主要分为两大流派，Boosting一族通过将弱学习器提升为强学习器的集成方法来提高预测精度（典型算法为AdaBoost，详见[Learn R | AdaBoost of Data Mining](https://zhuanlan.zhihu.com/p/24790439?refer=The-Art-of-Data)）；而另一类则为Bagging，即通过自助采样的方法生成众多并行式的分类器，通过“少数服从多数”的原则来确定最终的结果（典型算法为随机森林，详见[Learn R | Random Forest of Data Mining](https://zhuanlan.zhihu.com/p/24349592?refer=The-Art-of-Data)）。   

今天所要学习的GBDT同样是属于Boosting大家庭中的一员，自算法的诞生之初，它就和SVM一起被认为是泛化能力（generalization）较强的算法。近些年来更因为被用于构建搜索排序的机器学习模型而引起广泛的关注。

>除此之外，GBDT还是目前竞赛中最为常用的一种机器学习算法，因为它不仅可以适用于多种场景，而且相比较于其他算法还有着出众的准确率，如此优异的性能也让GBDT收获了机器学习领域的“屠龙刀”这一赞誉。    

那么，如此优秀的算法在实际中是如何进行工作的呢？背后的基本原理又是什么？接下来，我们就来走进算法内部，揭开这把“屠龙宝刀”的神秘面纱。    

## 一、GBDT之DT——回归树    
GBDT主要由三个概念组成：Regression Decistion Tree、Gradient Boosting与Shrinkage，只有弄清楚这三个概念，我们才能明白算法的基本原理。首先来学习第一个概念：Regression Decistion Tree，即回归决策树。

提到决策树，相信很多人会潜意识的想到最常见的分类决策树（ID3、C4.5、CART等等），但要把GBDT中的DT也理解为分类决策树那就大错特错了。实际上，决策树不仅可以用于分类，还可用于回归，它的作用在于数值预测，例如明天的温度、用户的年龄等等，而且对基于回归树所得到的数值进行加减是有意义的（例如10岁+5岁-3岁=12岁），这是区别于分类树的一个显著特征（毕竟男+女=是男是女?，这样的运算是毫无道理的）。GBDT在运行时就使用到了回归树的这个性质，它将累加所有树的结果作为最终结果。所以，GBDT中的所有决策树都是回归树，而非分类树。

接着，我们对问题进行进一步细分，来分析具体的一棵回归树的运行流程。

作为对比，简要回顾下分类树的运行过程：以ID3为例，穷举每一个属性特征的信息增益值，每一次都选取使信息增益最大的特征进行分枝，直到分类完成或达到预设的终止条件，实现决策树的递归构建。    

回归树的运行流程与分类树基本类似，但有以下两点不同之处：

- 第一，回归树的每个节点得到的是一个预测值而非分类树式的样本计数，假设在某一棵树的某一节点使用了年龄进行分枝（并假设在该节点上人数>1），那么这个预测值就是属于这个节点的所有人年龄的平均值。
- 第二，在分枝节点的选取上，回归树并没有选用最大熵值来作为划分标准，而是使用了最小化均方差，即$\frac{\sum_{i=1}^{n} (x_i-\bar{x} )^2}{n}$ 。这很好理解，被预测出错的次数越多，错的越离谱，均方差就越大，通过最小化均方差也就能够找到最靠谱的分枝依据。

>一般来讲，回归树的分枝不太可能实现每个叶子节点上的属性值都唯一，更多的是达到我们预设的终止条件即可（例如叶子个数上限），这样势必会存在多个属性取值，那么该节点处的预测值自然就为基于这些样本所得到的平均值了。   

## 二、GBDT之GB——梯度提升
在简单了解了回归树后，继续来看第二个概念：梯度提升（Gradient Boosting）。

首先需要明确，GB本身是一种理念而非一个具体的算法，其基本思想为：沿着梯度方向，构造一系列的弱分类器函数，并以一定权重组合起来，形成最终决策的强分类器（由于梯度提升的具体内容与数学推导有一些复杂，而且即使不了解这块知识的来龙去脉也不妨碍对算法本身的理解，出于这样的考虑，本文将不涉及'Gradient'，梯度提升的内容也将放在下一篇文章中详细介绍）。


那么这一系列的弱分类器是怎么样形成的呢？这就是GBDT的核心所在：每一棵树所学习的是之前所有树结论和的残差，这个残差就是一个加预测值后能得真实值的累加量。

>举一个简单的例子，同样使用年龄进行分枝，假设我们A的真实年龄是18岁，但第一棵树的预测年龄是12岁，即残差为6岁。那么在第二棵树里我们把A的年龄设为6岁去学习，如果第二棵树真的能把A分到6岁的叶子节点，那累加两棵树的结论就是A的真实年龄；如果第二棵树的结论是5岁，则A仍然存在1岁的残差，第三棵树里A的年龄就变成1岁……以此类推学习下去，这就是梯度提升在GBDT算法中的直观意义。    

## 三、GBDT算法的简单应用
接下来还是通过训练一个用于预测年龄的模型来展现算法的运行流程（本节的内容与图片引用自博客文章[GBDT（MART） 迭代决策树入门教程 | 简介](https://blog.csdn.net/w28971023/article/details/8240756)）

首先，训练集有4个人A、B、C、D，他们的年龄分别是14、16、24、26。其中A，B分别是高一和高三学生；C，D分别是应届毕业生和工作两年的员工，可用于分枝的特征包括上网时长、购物金额、上网时段和对百度知道的使用方式等。如果是用一棵传统的回归决策树来训练，会得到如下图所示结果：   
<img src=./pictures/gbdt_1.jpg>   

但如果是用GBDT来做这件事，由于数据太少，我们限定叶子节点做多有两个，即每棵树都只有一个分枝，并且限定只学两棵树。我们会得到如下图所示结果：   
<img src=./pictures/gbdt_2.jpg>   

第一棵树的分枝与之前一样，也是使用购物金额进行区分，两拨人各自用年龄均值作为预测值，得到残差值-1、1、-1、1，然后拿这些残差值替换初始值去训练生成第二棵回归树，如果新的预测值和残差相等，则只需把第二棵树的结论累加到第一棵树上就能得到真实年龄了。

第二棵树只有两个值1和-1，直接可分成两个节点。此时所有人的残差都是0，即每个人都得到了真实的预测值。

将两棵回归树预测结果进行汇总，解释如下：

- A：14岁高一学生；购物较少；经常问学长问题；预测年龄A = 15 – 1 = 14
- B：16岁高三学生；购物较少；经常被学弟问问题；预测年龄B = 15 + 1 = 16
- C：24岁应届毕业生；购物较多，经常问师兄问题；预测年龄C = 25 – 1 = 24
- D：26岁工作两年员工；购物较多，经常被师弟问问题；预测年龄D = 25 + 1 = 26       

对比初始的回归树与GBDT所生成的回归树，可以发现，最终的结果是相同的，那我们为什么还要使用GBDT呢？

答案就是对模型过拟合的考虑。过拟合是指为了让训练集精度更高，学到了很多“仅在训练集上成立的规律”，导致换一个数据集后，当前规律的预测精度就不足以使人满意了。毕竟，在训练精度和实际精度（或测试精度）之间，后者才是我们想要真正得到的。    

>在上面这个例子中，初始的回归树为达到100%精度使用了3个特征（上网时长、时段、网购金额），但观察发现，分枝“上网时长>1.1h”很显然过拟合了，不排除恰好A上网1.5h, B上网1小时，所以用上网时间是不是>1.1小时来判断所有人的年龄很显然是有悖常识的。
而在GBDT中，两棵回归树仅使用了两个特征（购物金额与对百度知道的使用方式）就实现了100%的预测精度，其分枝依据更合乎逻辑（当然这里是相比较于上网时长特征而言），算法在运行中也体现了“如无必要，勿增实体”的奥卡姆剃刀原理。    

## 四、GBDT之Shrinkage——缩减
>Shrinkage是GBDT的第三个基本概念，中文含义为“缩减”。它的基本思想就是：每次走一小步逐渐逼近结果的效果，要比每次迈一大步很快逼近结果的方式更容易避免过拟合。换句话说缩减思想不完全信任每一个棵残差树，它认为每棵树只学到了真理的一小部分，累加的时候只累加一小部分，只有通过多学几棵树才能弥补不足。
Shrinkage仍然以残差作为学习目标，但由于它采用的是逐步逼近目标的方式，导致各个树的残差是渐变的而不是陡变的。之所以这样做也是基于模型过拟合的考虑（更为详细的内容可参考文末给出的参考资料）。

GBDT的基本内容大致介绍完毕，在下一文中，我们将从梯度的角度对算法进行推导与学习。    

References：

1. [GBDT（MART） 迭代决策树入门教程](https://blog.csdn.net/w28971023/article/details/8240756)
2. [GBDT：梯度提升决策树](https://www.jianshu.com/p/005a4e6ac775)     
3. [机器学习算法中GBDT与Adaboost的区别与联系是什么？](https://www.jianshu.com/p/005a4e6ac775)    
4. [梯度提升树(GBDT)原理小结 - 刘建平Pinard - 博客园](http://www.cnblogs.com/pinard/p/6140514.html)    
5. [[Machine Learning & Algorithm] 决策树与迭代决策树（GBDT）](http://www.cnblogs.com/maybe2030/p/4734645.html)