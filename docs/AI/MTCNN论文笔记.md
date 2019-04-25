<!-- 
---
title: "MTCNN论文笔记"    
author:     
date: Sep 11, 2018    

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
# <center>MTCNN论文笔记</center>
[Joint Face Detection and Alignment using Multi-task Cascaded Convolutional Networks](https://kpzhang93.github.io/MTCNN_face_detection_alignment/index.html)    

借鉴VJ人脸识别器的思想，设计了一个级联的CNN网络，将人脸检测和人脸对齐两个任务合并一起处理。设计的网络总共有三步：   
- Stage 1: 通过浅层CNN网络(P-Net)快速产生候选窗口   
- Stage 2: 通过稍复杂的CNN网络(R-Net)排除大部分非人脸窗口   
- Stage 3: 最后通过表达能力更强的网络(O-Net)获取最终的人脸窗口以及人脸的5个标注点(left eye, right eye, nose, left mouth corner, and right mouth
corner)      

## 网络整体结构    
<div align=center><img src=./pictures/MTCNN_1.png   />   </div>  
>Note: figure is copy from the origin paper

## input     
原始的输入图像reshape  size得到图像金字塔，作为后续的网络的输入    

## Stage1     
网络的第一步是一个人脸二分类任务，对于一个给定的样本 $x_i$,采用cross-entropy作为学习目标，    
$$L_{i}^{det} = -(y_{i}^{det} log(p_i) + (1-y_{i}^{det})(1- log(p_i)))$$

$p_i$是网络预测的样本$x_i$是人脸的概率，$y_{i}^{det} \in \{0,1\}$是真实标签。     


## CNN Architectures     
卷积网络结构上做了两点改动，    
- 减少卷积核数量，卷积核尺寸减小到 3 x 3
- 采用PReLU作为激活函数



## Stage2    
网络第二步是一个回归任务，对于每一个候选窗口，需要预测出人脸框的四个坐标点，这里采用的是预测值与真实值的欧式距离作为学习目标，对每一个学习样本$x_i$,    
$$L_{i}^{box}=||\hat{y}_{i}^{box} - y_{i}^{box}||_{2}^{2}$$    
$\hat{y}_{i}^{box}$是网络预测值。    

## Stage3     
第三步和第二步相似，也可作为一个回归任务，    
$$L_{i}^{Landmark}=||\hat{y}_{i}^{Landmark} - y_{i}^{Landmark}||_{2}^{2}$$  
$\hat{y}_{i}^{Landmark}$ 是人脸的坐标标注。


## 多任务学习    
由于上述的三个损失函数的输入图像的类型不相同，所以在某些情况下，部分损失函数不起作用，如输入的是背景区域时，只有 $L_{i}^{det}$需要计算，后面的两个损失函数直接置为0，所以整体的损失函数可以形式化的表述为：      
$$min \sum_{i=1}^{N} \sum_{j \in \{det,box,landmark\}} \alpha_j \beta_{i}^{j}  L_{i}^{j} $$    
$N$是训练样本数，$\alpha_j$是权重系数，在第一和第一步中采用的是$\alpha_{det}=1,\alpha_{box}=0.5,\alpha_{landmark}=0.5$，在第三步采用的是$\alpha_{det}=1,\alpha_{box}=0.5,\alpha_{landmark}=1$。 $\beta_{i}^{j} \in \{0,1\}$在三步中一样。    

## Online Hard sample mining    
具体的是在所有网络前向传播的时候，计算所有样本的损失，然后对损失值排序，取前70%，作为$hard$ $samples$，只在这部分样本上计算梯度，作反向传播.    

## non-maximum suppression(NMS)     
非极大值抑制，这个操作的主要目的是去掉冗余的候选框。在第一步(以及第二步)的网络中，会产生大量的候选框，但是这些候选框之间有包含或者交叉的情况，被包含的候选框以及和其他候选框交叉但是没有完整的覆盖一张人脸的候选框可以过滤掉，减小后续计算量。具体方法如下：     
假设有6个矩形框，按分数(矩形框面积大小,或者第一步的网络预测出的是人脸的概率)排序，假设从小到大依次为 A、B、C、D、E、F。     
- 从分数最大的F开始，分别判断A~E与F的重叠度IOU 是否大于某个给定的阈值    
- 假设B、D与F的重叠度超过阈值，那么就扔掉B、D，并标记F是保留项    
- 从剩下的A、C、E中，选择分数最大的E，类似上面的操作，计算A、C与E的重叠度，超过阈值就扔掉，标记E是保留项     
- 重复这个操作，直到找到所有保留框      




[非极大值抑制（Non-Maximum Suppression，NMS）](https://www.cnblogs.com/makefile/p/nms.html)