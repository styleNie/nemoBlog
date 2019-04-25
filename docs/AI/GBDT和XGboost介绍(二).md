
<!-- ---
title: "GBDT和XGBoost介绍（二）"    
author:     
date: February 18, 2019    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
---
-->
# <center>GBDT和XGBoost介绍（二）</center>  
 

>转自https://zhuanlan.zhihu.com/p/25805870 

## 一、GBDT的梯度提升过程    
在上一文中，我们详细讲述了GBDT算法的基本概念，并通过一个简单的小例子阐述它在实际使用中的运行流程，可以说我们对算法已经有了一个初步的认识。不过还留有一个问题尚待解决，那就是如何理解GBDT中的Gradient（梯度）？

众所周知，基于Boosting的集成学习是通过迭代得到一系列的弱学习器，进而通过不同的组合策略得到相应的强学习器。在GBDT的迭代中，假设前一轮得到的强学习器为$f_{t-1}(x)$，对应的损失函数则为$L(y,f_{t-1}(x))$。因此新一轮迭代的目的就是找到一个弱学习器$h_t(x)$，使得损失函数$L(y,f_{t-1}(x)+h_t(x))$达到最小。

因此问题的关键就在于对损失函数的度量，这也正是难点所在，毕竟损失函数多种多样，怎么样才能找到一种通用的拟合方法呢？    
<img src=./pictures/gbdt_3.jpg>    

(上图为常用的损失函数及其梯度，图片来源: [The Elements of Statistical Learning](https://web.stanford.edu/~hastie/ElemStatLearn//) ，下同)    

针对这一问题，机器学习界的大牛Freidman提出了梯度提升算法：利用最速下降的近似方法，即利用损失函数的负梯度在当前模型的值，作为回归问题中提升树算法的残差的近似值，拟合一个回归树。

这样的话，第t轮的第i个样本的负梯度表示为：$r_{ti}=-\left[ \frac{\partial L(y,f(x_i))}{\partial f(x_i)} \right]_{f(x)=f_{t-1}(x)}$  ，算法的完整流程如下：

<img src=./pictures/gbdt_4.jpg>    

接下来对上图中的算法步骤进行详细解释：

- 初始化弱学习器$f_0(x)$，得到使损失函数极小化的一个常数值，此时树仅有一个根节点
- 计算损失函数的负梯度值，以此作为残差的估计
  - 针对选取的不同的损失函数（平方、绝对值、Huber），对应图1中不同的梯度值；
  - 算法中嵌套两层循环，分别为迭代轮数$m$和样本$i$

- 利用计算得到的$(x_i,r_{ti})$拟合一棵CART回归树，得到第m轮的回归树，对应的叶子节点区域为$R_{jm},j=1,2,...,J_m（J_m为回归树t的叶子节点的个数）$
- 接着，对叶子区域计算最佳拟合值$\gamma _{jm}$（损失函数极小化）并更新强学习器$f_m(x)$
- 最后，在迭代结束后输出最终模型$\widehat{f}(x)$    

## 二、GBDT小结
至此，GBDT的内容就基本讲完了，对这个算法，一方面我们可以从残差的角度来理解，每一棵回归树都是在学习之前的树的残差；另一方面也可以从梯度的角度掌握算法，即每一棵回归树通过梯度下降法学习之前的树的梯度下降值。

这样看来，这两种理解角度从总体流程和输入输出上没有区别的，它们都是迭代回归树，都是累加每棵树结果作为最终结果，每棵树都在学习前面树尚存的不足。而不同之处就在于每一步迭代时的求解方法的不同，前者使用残差（残差是全局最优值），后者使用梯度（梯度是局部最优方向），简单一点来讲就是前者每一步都在试图向最终结果的方向优化，后者则每一步试图让当前结果更好一点。

>看起来前者更科学一点，毕竟有绝对最优方向不学，为什么舍近求远去估计一个局部最优方向呢？原因在于灵活性。前者最大问题是，由于它依赖残差，损失函数一般固定为反映残差的均方差，因此很难处理纯回归问题之外的问题。而后者求解方法为梯度下降，只要可求导的损失函数都可以使用。    

最后小结一下GBDT算法的优缺点。

优点：

- 预测精度高
- 适合低维数据
- 能处理非线性数据    

缺点：

- 并行麻烦（因为上下两棵树有联系）
- 如果数据维度较高时会加大算法的- 计算复杂度

## 三、GBDT算法的R实现
在R中，我们可以使用gbm包中的相关函数来实现算法，首先进行安装与载入。
```r
> install.packages('gbm')
> library(gbm)  
```  

```r
# gbm()函数的调用公式及主要参数如下：
> gbm(formula = formula(data),distribution = "bernoulli",data = list(),
+ n.trees = 100,shrinkage = 0.001,...,
+ bag.fraction = 0.5,interaction.depth = 1)
# distribution:损失函数的形式,可选择 "gaussian"(squared error),"laplace" (absolute loss),"bernoulli"(logistic regression for 0-1 outcomes), "huberized"(huberized hinge loss for 0-1 outcomes)等
# n.trees:迭代次数
# shrinkage:学习速率,我们都知道步子迈得太大容易扯着蛋，所以学习速率是越小越好，但是步子太小的话，步数就得增加，也就是训练的迭代次数需要加大才能使模型达到最优，这样训练所需时间和计算资源也相应加大了,gbm作者的经验法则是设置参数在0.01-0.001之间
# bag.fraction:再抽样比率
# 除此之外,函数中还有其他参数,可自行查阅相关文档     
```   

接下来选用TH.data包中的bodyfat数据集进行实证分析，这个数据集记录了71名健康女性的身体数据，包括年龄、腰围、臀围等变量，用于身体脂肪（DEXfat）的预测分析。   
```r
> library(TH.data)
> data(bodyfat)
> dim(bodyfat)
[1] 71 10
> head(bodyfat)
   age DEXfat waistcirc hipcirc elbowbreadth kneebreadth anthro3a anthro3b
47  57  41.68     100.0   112.0          7.1         9.4     4.42     4.95 
48  65  43.29      99.5   116.5          6.5         8.9     4.63     5.01 
49  59  35.41      96.0   108.5          6.2         8.9     4.12     4.74     
50  58  22.79      72.0    96.5          6.1         9.2     4.03     4.48     
51  60  36.42      89.5   100.5          7.1        10.0     4.24     4.68     
52  61  24.13      83.5    97.0          6.5         8.8     3.55     4.06       
   anthro3c anthro4
47     4.50    6.13
48     4.48    6.37
49     4.60    5.82
50     3.91    5.66
51     4.15    5.91
52     3.64    5.14    
```   

```r
# 模型构建
> gbdt_model <- gbm(DEXfat~.,distribution = 'gaussian',data=bodyfat,
+ n.trees=1000,shrinkage = 0.01)
```   

```r
# 使用gbm.pref()确定最佳迭代次数
> best.iter <- gbm.perf(gbdt_model)
> best.iter
[1] 399
```    

<img src=./pictures/gbdt_5.jpg>  

```r
# 查看各变量的重要程度
> summary.gbm(gbdt_model,best.iter)
                      var   rel.inf
hipcirc           hipcirc 36.276643
waistcirc       waistcirc 31.447718
anthro3c         anthro3c  6.561716
anthro4           anthro4  6.447923
anthro3b         anthro3b  6.423434
anthro3a         anthro3a  5.068798
kneebreadth   kneebreadth  3.834679
age                   age  2.050727
elbowbreadth elbowbreadth  1.888361
```    

<img src=./pictures/gbdt_6.jpg>   

```r
# 查看各变量的边际效应,用数值来自定义变量查询
> par(mfrow=c(1,3))
> plot.gbm(gbdt_model,3,best.iter)
> plot.gbm(gbdt_model,4,best.iter)
> plot.gbm(gbdt_model,5,best.iter)
```    

<img src=./pictures/gbdt_7.jpg> 

```r
# 使用建立好的模型进行DEXfat值的预测
> gbdt_fit <- predict(gbdt_model,bodyfat,best.iter)
> head(gbdt_fit)
[1] 43.78154 43.51006 36.23172 23.54703 36.71636 22.47789
# 计算预测结果与真实结果的方差值
> print(sum((bodyfat$DEXfat-gbdt_fit)^2))
[1] 924.0749
```

这就是GBDT算法在R中的一个实例应用，由于我们的目的在于获取所需的预测值，所以无法直接通过对比预测值与实际值的一致程度（即预测精度）来判断模型的好坏，而是选用了方差作为衡量模型优劣的标准。如果我们想要查看一个模型在新的数据集上的表现如何，可以考虑生成训练数据集与测试数据集，操作如下：    
```r
> index <- sample(2,nrow(bodyfat),replace = TRUE,prob=c(0.7,0.3))
> traindata <- bodyfat[index==1,]
> testdata <- bodyfat[index==2,]
```   

这样的话，生成模型选用traindata，进行预测可选用testdata。

最后需要注意的是，在bodyfat的案例中，我们一开始选用了回归中最常见的平方损失，但要知道不同的损失函数对最终的预测精度也会起到不同的影响，其他参数也是同理。所以，如果要对模型进行进一步改进的话，参数选择这块还是大有文章可做的。



## References：

1. [GBDT（Gradient Boost Decision Tree）](http://www.cnblogs.com/zhizhan/p/5088775.html)   
2. [GBDT：梯度提升决策树](https://www.jianshu.com/p/005a4e6ac775)
3. [梯度提升树(GBDT)原理小结 - 刘建平Pinard - 博客园](http://www.cnblogs.com/pinard/p/6140514.html)
4. [gbm function | R Documentation](http://www.rdocumentation.org/packages/gbm/versions/2.1.1/topics/gbm)
5. [Gradient Boosting Machine (GBM) Tutorial](http://allstate-university-hackathons.github.io/PredictionChallenge2016/GBM)
6. [机器学习（四）--- 从gbdt到xgboost](http://www.cnblogs.com/mfryf/p/6276921.html)
7. [梯度下降与随机梯度下降 - 博客频道 - CSDN.NET](http://blog.csdn.net/u014568921/article/details/44856915)
8. [一步一步理解GB、GBDT、xgboost - wxquare - 博客园](http://www.cnblogs.com/wxquare/p/5541414.html)
