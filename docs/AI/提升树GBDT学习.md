<!-- 
---
title: "提升树GBDT学习"    
author:     
date: March 05, 2018    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false    
---
-->
# <center>提升树GBDT学习</center>  
   

## 提升树 (boosting tree)   
提升方法采用加法模型(基函数的线性组合)与前向分步算法,以决策树为基函数的提升方法成为提升树。对分类问题决策树是二叉分类树，对回归问题决策树是二叉回归树.   
提升树模型可以表示为决策树的加法模型：   

$$ f_{M}(x) = \sum_{m=1}^MT(x;\Theta_{m}) $$   

其中，$T(x;\Theta_{m})$ 表示决策树；$\Theta_{m}$ 为决策树的参数；$M$为树的个数。   

