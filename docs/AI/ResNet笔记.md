<!-- 
---
title: "ResNet笔记"    
author:     
date: Sep 12, 2018    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: false     
  ignoreLink:false    
  
html:    
  embed_local_images: true    
  embed_svg: true    
  offline: false    
  toc: true    

print_background: false    

export_on_save:    
  html: true    

---
-->
# <center>ResNet笔记</center>
  

[Deep Residual Learning for Image Recognition](https://arxiv.org/abs/1512.03385)      

深度神经网络的成功似乎给了人们一种错觉：网络越深，则网络学习效果越好。但是实际使用中人们发现，网络加深后除了训练难度加大，网络在训练过程中的准确率随着训练次数的加大出现了下降(Degradation problem)。针对这个现象，作者做了仔细分析，并提出了一种新的结构来避免这种现象，使得网络层数可以任意加大而不会出现训练准确率下降的问题。作者设计了一个152层的深度残差网络，在ImageNet网络竞赛中取得了第一名的成绩。      


## 残差网络结构     
考虑一个浅层网络，然后在上面逐渐加恒等映射层(identity mapping),即新增加的层只是复制浅层网络学习到的特征，如果这个构造网络的解存在的话，那么表明，一个深层网络的训练错误率应该不会比浅层网络的训练错误率更高。但是现在的实验表明现有的训练方法很难去找到这样的解，所以一定是训练方法存在问题。     

传统的深度网络的每一层学习的是前一层的输出，残差网络的每一层学习的前一层的残差。用$H(x)$表示网络正常的输出，新增加的层  $f(x)=H(x) -x$,所以原始的输入层的期待输入的是 $F(x)+x$。    

 $F(x)+x$可以用```shortcut connections```来实现。     

 <div align=center><img src=./pictures/ResNet_1.png />   </div>     

 文章介绍的第一种 ```building block```如上图所示，$x$为输入，$y$为输出，$F(x,\{W_i\})$表示需要学习的残差映射，    
 $$y=F(x,\{W_i\}) + x $$    
 图2中有两层，$F=W_2 \sigma(W_1 x)$,$\sigma$为ReLU 函数，执行$F+x$时，必须保证$F$与$x$的维度一致，可以对$x$做一个线性投影来保证维度一致，所以     
 $$y=F(x,\{W_i\}) + W_s x$$     


 ```building block```的层数可以根据需要增加，如下图所示    
  <div align=center><img src=./pictures/ResNet_2.png />   </div>  