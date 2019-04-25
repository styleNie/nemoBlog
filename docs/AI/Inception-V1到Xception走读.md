<!-- 
---
title: "Inception-V1 到 Xception走读"    
author:     
date: Sep 17, 2018    

toc:     
  depth_from: 1    
  depth_to: 6    
  ordered: false    
  ignoreLink:false    
  
html:    
  embed_local_images: true    
  embed_svg: true    
  offline: false    
  toc: false    

print_background: false    

export_on_save:    
  html: true    

---
-->
# <center>Inception-V1 到 Xception走读</center>    

## GoogLeNet      
[Going Deeper with Convolutions](https://arxiv.org/abs/1409.4842)      
自AlexNet、VGGNet在图像任务中取得成功后，增加网络的深度与宽度以取得更好的效果似乎成为了最直观、简便的方法，但是这种最简单的方法带来了两个不可忽视的问题：   
- 网络更深更宽之后，网络的参数个数也增加，网络也更容易过拟合    
- 计算量与计算资源的增大     

作者从增加网络稀疏性入手，提出了Inception architecture，并在此基础上构建GoogleNet.     

### Inception architecture    
<div align=center><img src=./pictures/GoogleNet_1.png /></div>  

Inception module的设计主要的是在单个卷积层中使用不同大小的卷积核，增强网络的表达能力。    


## Inception-V2     
[Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift](https://arxiv.org/abs/1502.03167)

Inception-V2在Inception-V1的基础上做了两点改动：    
- 每一层使用 batch Normalization,减小网络的每一层内部的 Covariate Shift，加速网络训练        
- 用两个3x3的卷积替代5x5的卷积，降低了网络的参数个数，也加速了计算      



## Inception-V3     
[Rethinking the Inception Architecture for Computer Vision](https://arxiv.org/abs/1512.00567)     

### General Design Principles    
- 避免表示瓶颈，特征图的大小应缓慢下降，特别是在网络开始阶段          
- 高维表示更容易处理，增加网络的每一层的激活度会得到更多的有区分度的特征，会加快训练     
- 在低维嵌入空间上做空间汇聚不会带来太多的信息损失；比如在进行3x3的卷积之前，可以对输入做降维，而不会有严重的后果     
- 平衡网络的深度和宽度     

### 具体改动     
- 大卷积分解为小卷积     
- nxn的卷积分解为nx1和1xn的卷积    
- 附加分类器，在网络的中间层加上分类器，计算与梯度，将其加到网络的常规梯度上，再执行反向传播，有效避免了梯度消失的问题     
- 降低feature map的尺寸，图9为常规的降尺寸的方法，图10为文章中提出的降尺寸的方法     

<div align=center><img src=./pictures/InceptionV3_1.png /></div> 

<div align=center><img src=./pictures/InceptionV3_2.png /></div> 
      


## Inception-V4     
[Inception-v4, Inception-ResNet and the Impact of Residual Connections on Learning](https://arxiv.org/abs/1602.07261)     
Inception-V4主要是将Inception与ResNet结合起来，并且网络深度加大。 具体的网络结构见原文，原文中有详细的网络结构图。       
<div align=center><img src=./pictures/InceptionV3_3.png /></div> 
<div align=center><img src=./pictures/InceptionV3_4.png /></div>
<div align=center><img src=./pictures/InceptionV3_5.png /></div>  
<div align=center><img src=./pictures/InceptionV3_6.png /></div>  


## Xception     
[Xception: Deep Learning with Depthwise Separable Convolutions](https://arxiv.org/abs/1610.02357)     

Xception在Inception-V4的基础上用depthwise separable convolutions替换了原来的常规的卷积操作。由经典的 Inception-V3 module触发，得到一个简化版的Inception结构      
<div align=center><img src=./pictures/InceptionV3_7.png /></div>      
<div align=center><img src=./pictures/InceptionV3_8.png /></div>  

然后进一步延伸，将input用1x1卷积得到输出作为3x3卷积的输入，每个3x3的卷积只和输入的一部分做卷积，    
<div align=center><img src=./pictures/InceptionV3_9.png /></div>      
再延伸，保持3x3卷积的个数和1x1卷积的输出通道数一致，也即每个3x3卷积和1个输入通道做卷积。      
<div align=center><img src=./pictures/InceptionV3_10.png /></div>     

这个结构和depthwise separable convolutions非常相似了。两者间有两点不一样：     
- 顺序不一样，depthwise separable convolution先进行channel-wise spatial convolution 再执行 1x1 的卷积，图4是先1x1的卷积，然后执行channel-wise spatial convolution     
- 图4中每个操作都添加了一个ReLu的非线性激活函数，但是在depthwise separable convolutions中没有     

于是将depthwise separable convolution引入到Inception-V3就得到了Xception网络。     
网络具体结构参见原文。




Reference:     
[GoogLeNet 之 Inception(V1-V4)](https://blog.csdn.net/hejin_some/article/details/78636586)     
[从GoogLeNet至Inception v3](https://blog.csdn.net/Numeria/article/details/73611456)      
[深度学习之图像分类模型inception v2、inception v3解读](https://blog.csdn.net/sunbaigui/article/details/50807418)       
[Xception算法详解](https://blog.csdn.net/u014380165/article/details/75142710)    

