<!-- 
---
title: "GBDT和XGBoost介绍（三）"    
author:     
date: February 18, 2019    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
---
-->
# <center>GBDT和XGBoost介绍（三）</center>  
 

>转自https://zhuanlan.zhihu.com/p/25993218 

## 一、XGBoost简介
在GBDT的学习过程中，不少博客都提到了该算法的升级版——XGBoost，并对它赞赏有加。所以，这一块的知识（包括算法的基本内容，数学推导与案例实现）将使用两篇文章的篇幅来进行学习掌握。

经过前面的学习，我们已经知道，GBDT是一种基于集成思想下的Boosting学习器，并采用梯度提升的方法进行每一轮的迭代最终组建出强学习器，这样的话算法的运行往往要生成一定数量的树才能达到令我们满意的准确率。当数据集大且较为复杂时，运行一次极有可能需要几千次的迭代运算，这将对我们使用算法造成巨大的计算瓶颈。

针对这一问题，华盛顿大学的陈天奇博士开发出了XGBoost（eXtreme Gradient Boosting），它是Gradient Boosting Machine的一个c++实现，并在原有的基础上加以改进，从而极大地提升了模型训练速度和预测精度。可以说，XGBoost是Gradient Boosting的高效实现。    
>XGBoost最大的特点在于它能够自动利用CPU的多线程进行并行计算，同时在算法上加以改进提高了精度。在Kaggle的希格斯子信号识别竞赛中，XGBoost因为出众的效率与较高的预测准确度在比赛论坛中引起了参赛选手的广泛关注，在1700多支队伍的激烈竞争中占有一席之地。随着它在Kaggle社区知名度的提高，在其他的比赛中也有队伍借助XGBoost夺得第一。    

接下来学习XGBoost算法的数学推导过程。    

## 二、目标函数：损失与正则
在监督学习中，我们通常会构造一个目标函数和一个预测函数，使用训练样本对目标函数最小化学习到相关的参数，然后用预测函数和训练样本得到的参数来对未知的样本进行分类的标注或者数值的预测。在XGBoost中，目标函数的形式为：$Obj(\Theta) = L(\theta) + \Omega(\Theta)$

- $L(\theta)$：损失函数，常用损失函数有：
  - 平方损失：$L(\theta) = \sum_i (y_i-\hat{y}_i)^2$
  - Logistic损失：$L(\theta) = \sum_i[ y_i\ln (1+e^{-\hat{y}_i}) + (1-y_i)\ln (1+e^{\hat{y}_i})]$

- $\Omega(\Theta)$：正则化项，之所以要引入它是因为我们的目标是希望生成的模型能准确的预测新的样本（即应用于测试数据集），而不是简单的拟合训练集的结果（这样会导致过拟合）。所以需要在保证模型“简单”的基础上最小化训练误差，这样得到的参数才具有好的泛化性能。而正则项就是用于惩罚复杂模型，避免预测模型过分拟合训练数据，常用的正则有$L_1$正则与$L_2$正则。    
<img src=./pictures/gbdt_8.jpg>  

上图所展示的就是损失函数与正则化项在模型中的应用（图片来源：[Introduction to Boosted Trees](http://link.zhihu.com/?target=http%3A//xgboost.readthedocs.io/en/latest/model.html)）。观察发现，如果目标函数中的损失函数权重过高，那么模型的预测精度则不尽人意，反之如果正则项的权重过高，所生成的模型则会出现过拟合情况，难以对新的数据集做出有效预测。只有平衡好两者之间的关系，控制好模型复杂度，并在此基础上对参数进行求解，生成的模型才会“简单有效”（这也是机器学习中的偏差方差均衡）。

## 三、XGBoost的推导过程    
### 1. 目标函数的迭代与泰勒展开

由于之前已经学习过树的生成及集成方法，这里不再赘述。首先，我们可以把某一次迭代后集成的模型表示为：$\hat{y}_i = \sum_{k=1}^K f_k(x_i), f_k \in \mathcal{F}$（$\hat{y}_i$ 也就是上文中的$f_m(x)$）

相对应的目标函数：$\text{Obj}(\theta) = \sum_i^n l(y_i, \hat{y}_i) + \sum_{k=1}^K \Omega(f_k)$

将这两个公式进行扩展，应用在前t轮的模型迭代中，具体表示为： 
$$    
\begin{split}\hat{y}_i^{(0)} &= 0\\
\hat{y}_i^{(1)} &= f_1(x_i) = \hat{y}_i^{(0)} + f_1(x_i)\\
\hat{y}_i^{(2)} &= f_1(x_i) + f_2(x_i)= \hat{y}_i^{(1)} + f_2(x_i)\\
&\dots\\
\hat{y}_i^{(t)} &= \sum_{k=1}^t f_k(x_i)= \hat{y}_i^{(t-1)} + f_t(x_i)
\end{split}
$$

$\hat{y}_i^{(t-1)}$就是前$t-1$轮的模型预测，$f_t(x_i)$为新$t$轮加入的预测函数。    

这里自然就涉及一个问题：如何选择在每一轮中加入的$f(x_i)$呢？答案很直接，选取的$f(x_i)$必须使得我们的目标函数尽量最大地降低（这里应用到了Boosting的基本思想，即当前的基学习器重点关注以前所有学习器犯错误的那些数据样本，以此来达到提升的效果）。先对目标函数进行改写，表示如下：   
$$
\begin{split}\text{Obj}^{(t)} & = \sum_{i=1}^n l(y_i, \hat{y}_i^{(t)}) + \sum_{i=1}^t\Omega(f_i) \\
          & = \sum_{i=1}^n l(y_i, \hat{y}_i^{(t-1)} + f_t(x_i)) + \Omega(f_t) + constant
\end{split}
$$    

如果我们考虑使用平方误差作为损失函数，公式可改写为：
$$
\begin{split}\text{Obj}^{(t)} & = \sum_{i=1}^n (y_i - (\hat{y}_i^{(t-1)} + f_t(x_i)))^2 + \sum_{i=1}^t\Omega(f_i) \\
          & = \sum_{i=1}^n [2(\hat{y}_i^{(t-1)} - y_i)f_t(x_i) + f_t(x_i)^2] + \Omega(f_t) + constant
\end{split}
$$    

更加一般的，对于不是平方误差的情况，我们可以采用如下的泰勒展开近似来定义一个近似的目标函数，方便我们进行这一步的计算。    

>泰勒展开：$f(x+\Delta  x)\simeq f(x)+f^{'}(x)\Delta x+\frac{1}{2} f^{''}(x)\Delta x^2$    

$$
\text{Obj}^{(t)} = \sum_{i=1}^n [l(y_i, \hat{y}_i^{(t-1)}) + g_i f_t(x_i) + \frac{1}{2} h_i f_t^2(x_i)] + \Omega(f_t) + constant
$$    

其中$g_i= \partial_{\hat{y}_i^{(t-1)}} l(y_i, \hat{y}_i^{(t-1)})，h_i = \partial_{\hat{y}_i^{(t-1)}}^2 l(y_i, \hat{y}_i^{(t-1)})$    

>如果移除掉常数项，我们会发现这个目标函数有一个非常明显的特点，它只依赖于每个数据点的在误差函数上的一阶导数和二阶导数（$\sum_{i=1}^n [g_i f_t(x_i) + \frac{1}{2} h_i f_t^2(x_i)] + \Omega(f_t)$）。有人可能会问，这个公式似乎比我们之前学过的决策树学习难懂。为什么要花这么多力气来做推导呢？
这是因为，这样做会使我们可以很清楚地理解整个目标是什么，并且一步一步推导出如何进行树的学习。这一个抽象的形式对于实现机器学习工具也是非常有帮助的。因为它包含所有可以求导的目标函数，也就是说有了这个形式，我们写出来的代码可以用来求解包括回归，分类和排序的各种问题，正式的推导可以使得机器学习的工具更加一般化。    

### 2. 决策树的复杂度

接着来讨论如何定义树的复杂度。我们先对于f的定义做一下细化，把树拆分成结构部分q和叶子权重部分w。其中结构函数q把输入映射到叶子的索引号上面去，而w给定了每个索引号对应的叶子分数是什么。具体公式为：$f_t(x) = w_{q(x)}, w \in R^T, q:R^d\rightarrow \{1,2,\cdots,T\} $

当我们给定上述定义后，那么一棵树的复杂度就为$\Omega(f) = \gamma T + \frac{1}{2}\lambda \sum_{j=1}^T w_j^2$    

>这个复杂度包含了一棵树里面节点的个数（左侧），以及每个树叶子节点上面输出分数的$L_2$模平方（右侧）。当然这不是唯一的一种定义方式，不过这一定义方式学习出的树效果一般都比较不错。    

简单提及一下$\gamma$ 和$\lambda$ 两个系数的作用，$\gamma$ 作为叶子节点的系数，使XGBoost在优化目标函数的同时相当于做了预剪枝；$\lambda$ 作为$L_2$平方模的系数也是要起到防止过拟合的作用。

这里举一个小例子加深对复杂度的理解（图片来源：[Introduction to Boosted Trees](http://link.zhihu.com/?target=http%3A//xgboost.readthedocs.io/en/latest/model.html)，下同）    
<img src=./pictures/gbdt_9.jpg>    

上图为实际生成的一棵决策树，底部的数字代表决策树的预测值，那么这棵树的复杂度自然就为：$\Omega =\gamma 3+\frac{1}{2} \lambda \sum_{j=1}^{T}{(4+0.01+1)}$    

### 3. 目标函数的最小化

接下来就是非常关键的一步，在这种新的定义下，我们可以把目标函数进行如下改写，其中I被定义为每个叶子上面样本集合$I_j = \{i|q(x_i)=j\}$    

$$
\begin{split}Obj^{(t)} &\approx \sum_{i=1}^n [g_i w_{q(x_i)} + \frac{1}{2} h_i w_{q(x_i)}^2] + \gamma T + \frac{1}{2}\lambda \sum_{j=1}^T w_j^2\\
&= \sum^T_{j=1} [(\sum_{i\in I_j} g_i) w_j + \frac{1}{2} (\sum_{i\in I_j} h_i + \lambda) w_j^2 ] + \gamma T
\end{split}
$$

分别定义$G_j = \sum_{i\in I_j} g_i$与$H_j = \sum_{i\in I_j} h_i$，上式简化为   
$${Obj}^{(t)} = \sum^T_{j=1} [G_jw_j + \frac{1}{2} (H_j+\lambda) w_j^2] +\gamma T   
$$    

由此，我们将目标函数转换为一个一元二次方程求最小值的问题（在此式中，变量为$w_j$，函数本质上是关于$w_j$的二次函数），略去求解步骤，最终结果如下所示：   
$$
w_j^\ast = -\frac{G_j}{H_j+\lambda}，\begin{split}\\
{Obj}^\ast = -\frac{1}{2} \sum_{j=1}^T \frac{G_j^2}{H_j+\lambda} + \gamma T
\end{split}
$$    

乍一看目标函数的计算与树的结构函数$q$没有什么关系，但是如果我们仔细回看目标函数的构成，就会发现其中$G_j$和$H_j$的取值都是由第$j$个树叶上数据样本所决定的。而第$j$个树叶上所具有的数据样本则是由树结构函数$q$决定的。也就是说，一旦树的结构$q$确定，那么相应的目标函数就能够根据上式计算出来。那么树的生成问题也就转换为找到一个最优的树结构$q$，使得它具有最小的目标函数。    

>计算求得的$Obj$代表了当指定一个树的结构的时候，目标函数上面最多减少多少。所有我们可以把它叫做结构分数(structure score)。    

<img src=./pictures/gbdt_10.jpg>   

上图为结构分数的一次实际应用，根据决策树的预测结果得到各样本的梯度数据，然后计算出实际的结构分数。正如图中所言，分数越小，代表树的结构越优。    

### 4. 枚举树的结构——贪心法

在前面分析的基础上，当寻找到最优的树结构时，我们可以不断地枚举不同树的结构，利用这个打分函数来寻找出一个最优结构的树，加入到我们的模型中，然后再重复这样的操作。不过枚举所有树结构这个操作不太可行，在这里XGBoost采用了常用的贪心法，即每一次尝试去对已有的叶子加入一个分割。对于一个具体的分割方案，我们可以获得的增益可以由如下公式计算得到：   

$$
Gain = \frac{1}{2} \left[\frac{G_L^2}{H_L+\lambda}+\frac{G_R^2}{H_R+\lambda}-\frac{(G_L+G_R)^2}{H_L+H_R+\lambda}\right] - \gamma
$$    

其中$\frac{G_L^2}{H_L+\lambda}$代表左子树分数，$\frac{G_R^2}{H_R+\lambda}$代表右子树分数，$\frac{(G_L+G_R)^2}{H_L+H_R+\lambda}$代表不分割时我们可以获得的分数，$\gamma$ 代表加入新叶子节点引入的复杂度代价。

对于每次扩展，我们还是要枚举所有可能的分割方案，那么如何高效地枚举所有的分割呢？假设需要枚举所有$x<a$这样的条件，那么对于某个特定的分割$a$我们要计算$a$左边和右边的导数和，在实际应用中如下图所示：   
<img src=./pictures/gbdt_11.jpg>   

我们可以发现对于所有的$a$，我们只要做一遍从左到右的扫描就可以枚举出所有分割的梯度与$G_L$和$G_R$。然后用上面的公式计算每个分割方案的分数就可以了。    

>但需要注意是：引入的分割不一定会使得情况变好，因为在引入分割的同时也引入新叶子的惩罚项。所以通常需要设定一个阈值，如果引入的分割带来的增益小于一个阀值的时候，我们可以剪掉这个分割。此外在XGBoost的具体实践中，通常会设置树的深度来控制树的复杂度，避免单个树过于复杂带来的过拟合问题。   

到这里为止，XGBoost的数学推导就简要介绍完毕。在下一文中，我们将去了解该算法相对于GBDT的优异特性以及学习XGBoost的R实现。


## References：

1. [Introduction to Boosted Trees](http://xgboost.readthedocs.io/en/latest/model.html)   

2. [XGBoost原理介绍](https://mp.weixin.qq.com/s?__biz=MzIyNjE2Nzk2Mw==&mid=2649623429&idx=1&sn=0793d85ccd73c833abec9415a6261946&chksm=f06e3671c719bf67243505bb2fb1ec8d42e4c8b7b147b84b4aa513ea67735ba9cc8fc85d48a1&mpshare=1&scene=24&srcid=0326vjycNNn9e4R1YY3QeKCi#rd)
3. [XGBoost与Boosted Tree](https://mp.weixin.qq.com/s?__biz=MzIzMDA1MTM3Mg==&mid=2653077581&idx=1&sn=6a68a87f6bd876420d303f25f7275c9d&chksm=f36f3b0ec418b218a2562bb0b27adbadd9401605c0bb3b147eafa5c046dcf3f574f7cc5d97c0&mpshare=1&scene=24&srcid=0326qLBu1retilXCIqvF7Vts#rd)
4. [xgboost：速度快效果好的Boosting模型](https://mp.weixin.qq.com/s?__biz=MzI1NDMyMjgyMA==&mid=2247483764&idx=1&sn=2cbaf871c9f4579cefe34b3805a22557&mpshare=1&scene=24&srcid=0326t6677DyczoesfFZwX9kl#rd)
5. [一步一步理解GB、GBDT、xgboost - wxquare - 博客园](http://www.cnblogs.com/wxquare/p/5541414.html)
6. [机器学习（四）--- 从gbdt到xgboost - 知识天地 - 博客园](http://www.cnblogs.com/mfryf/p/6276921.html)