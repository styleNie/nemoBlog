<!-- 
---
title: "SqueezeNext笔记"    
author:     
date: Sep 13, 2018    

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
# <center>SqueezeNext笔记</center>    
[SqueezeNext: Hardware-Aware Neural Network Design](https://arxiv.org/abs/1803.10615)  

SqueezeNext网络以SqueezeNet网络为基准，进行改进，网络参数仅为SqueezeNet的一半，但是保持准确率不下降。具体来说有以下改动：    
- 用two-stage squeeze module实现更激进的通道数减小，使得在 3x3卷积的时候使用的参数更少    
- 用可分离的3x3卷积进一步减小模型大小，在squeeze module后去掉了 1x1的分支    
- 使用逐个元素相加的shortcut connection，使得网络加深时，避免了梯度消失/爆炸问题(ResNet里的操作)           
- 在multi-processor embedded system上进行实验，并通过实验结果指导网络的设计，使网络inference时速度更快       

## SqueezeNext Design     
### Low Rank Filters   
假定输入 $x \in R^{HxWxC_i}$,输出$y \in R^{HxWxC_o}$,是用的卷积核为 $KxK$，则这一层的所有参数为 $K^2 C_i C_o$个。    
传统的训练好的网络中的权值参数存在着低秩特性，因此研究人员开始对初始化的权值添加低秩的限制，但是存在两个问题    
- 卷积核不低秩，无法实现压缩     
- 卷积核低秩，但是准确率下降，需要retraining     

针对这两个问题，作者设计用$1xK$和$Kx1$的卷积核替代$KxK$的卷积核，卷积核的参数个数也冲$K^2$下降到$2K$，同时也增加了网络深度。     

### Bottleneck Module      
借鉴SqueezeNet里减少通道数的方法，用一个 two-stage的squeez layer减少通道数。如图1中右图所示。     
<div align=center><img src=./pictures/SqueezeNext_1.png />   </div>       

### Fully Connected Layers    
在全连接之前，再加一个bottleneck layer,进一步减少通道数，从而减少全连接的参数。    


## 硬件模拟指导加速    
随后作者在硬件上部署了网络，测试性能，微调了网络结构。