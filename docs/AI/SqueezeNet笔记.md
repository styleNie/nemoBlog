<!-- 
---
title: "SqueezeNet笔记"    
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
# <center>SqueezeNet笔记</center>
  

[SqueezeNet: AlexNet-level accuracy with 50x fewer parameters and <0.5MB model size](https://arxiv.org/abs/1602.07360)  


ResNet网络提出之后，网络层数越来越多，因此训练网络花费的时间以及资源都大大增加，更重要的是模型越来越大，难以在移动硬件上部署，因此给网络"瘦身"成为了亟待解决的问题。具体研究路线上有两条，一是模型压缩，即压缩已经设计好的网络；二是设计参数更少的轻量级的网络。本文采取的是后一种思路。     

轻量级网络带来三个好处：    
- 在分布式训练平台上跨服务器通信量减小    
- 从云端加载模型到硬件上时需要的带宽减小    
- 更容易在FPGA等内存有限的设备上部署    


## SqueezeNet的设计策略     
- 用 1x1的卷积替代 3x3的卷积    
- 减少 3x3卷积的输入通道数     
- 延迟下采样，以便卷积层有更大的激活map     


## THE FIRE MODULE    
 <div align=center><img src=./pictures/SqueezeNet_1.png />   </div>      

图1中 $s_{1x1}$是squeeze层的所有 1x1卷积核的数目，$e_{1x1}$是expand层的所有 1x1卷积核的数目，$e_{3x3}$是expand层的所有 3x3卷积核的数目。在Fire Modules中令    
$$s_{1x1} < (e_{1x1} + e_{3x3})$$    
所以Squeeze层能限制输入到 3x3卷积核的通道数。    

## THE SQUEEZENET ARCHITECTURE    
 <div align=center><img src=./pictures/SqueezeNet_2.png />   </div>     
 图2是SqueezeNet的网络结构，中间和右边的两个增加了short connection.      


<div align=center><img src=./pictures/SqueezeNet_3.png />   </div>  
表3比较了使用Fire module和不用Fire module的参数个数，可以看到SqueezeNet的网络参数个数下降了一个数量级。但是后续的实验表明，准确率并没有显著的下降。     

<div align=center><img src=./pictures/SqueezeNet_4.png />   </div>  

