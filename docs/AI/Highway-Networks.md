<!-- 
---
title: "Highway Networks笔记"    
author:     
date: Sep 04, 2018    

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
# <center>Highway Networks笔记</center>
  

[Highway Networks](https://arxiv.org/abs/1505.00387)    

本文发表于2015年，在此之前，深度学习已经取得了极大的成功，并且大量的实验和研究文章都表明增大网络的层数，网络学习能力越强，可以获得更好的学习效果；但随之带来另一个问题，网络层数越多，越难以训练；对此，研究者提出了两种办法，一是设计合适的初始化权值，二是分阶段训练网络。本文作者受LSTM网络(Long Short Term Memory Recurrent neural network)中的“门”机制的启发，提出了一种新颖的结构，使得网络深度不再成为训练网络的瓶颈。作者将其命名为 Highway Networks。    

## Highway Networks     
经典的网络一般包含$L$层，在每一层中施加一个非线性变换 $H$ 到它的输入$x$上，得到输出$y$,$H$的参数用$W_H$表示，忽略偏置项，上述过程形式化的表述为      
$$y = H(x,W_H)$$    
在highway network中，增加了两项非线性变换，$T(x,W_T)$和$C(x,W_C)$,$T(x,W_T)$称为 ```transform gate```,$C(x,W_C)$称为```carry gate```,     
$$y=H(x,W_H) T(x,W_T) + x C(x,W_C)$$

这两项控制着输出项$y$中有多少信息是通过非线性变换得到，有多少信息是直接由输入项得到的。简单起见，文章中采用$C=1-T$,因此      
$$y=H(x,W_H) T(x,W_T) + x(1 - T(x,W_T))$$ 
上面的公式中 $x$,$y$,$H(x,W_H)$,$T(x,W_T)$的维数必须保持一致。但是有时需要改变尺寸，这里存在两种方法    
- 一是用采样或者填充零之后得到的$\hat x$替代$x$    
- 另一种是用plain layer(without highways)来改变维度，然后后面跟上堆叠的highway layers。本文中用的正是这种方法     

对于$H$和$T$变换都施加 权值共享和局部感受野操作。在Highway 层中，令$T(x)=\sigma(W_{T}^{T}x + b_T)$    

其中$\sigma(x)=\frac{1}{1+exp(-x)}$    

整个网络采用随机梯度下降法训练。