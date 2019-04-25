目标检测：R-CNN   
## 目标检测概述    
目标检测的基本任务是从一张图片中**定位**到目标并**识别**出目标的类别，即包含两个步骤：   
1)定位出目标在图片中的具体位置，一般用边框(bounding box)来表示     
2)识别出目标的类别。     

##传统的目标检测方法    
传统的目标检测方法框架大致如下：     
- 利用不同尺寸的窗口在图片上滑动，框住图片的某一部分作为候选区域    
- 设计图片特征提取方法，从候选区域中提取特征    
- 将提取的图片特征输入到分类器中     

不同的检测方法主要区别在于图片特征提取方法和分类器的选择上。本文的改动也集中在第二点，即特征提取上。      

## R-CNN    
<img src=./pictures/fast_rcnn_3.png >   

本文的网络框架图如上图所示，
- 从输入图像中提取约2000张候选区域     
- 将提取的2000张候选图像，逐一输入到卷积神经网络中提取出一个4096维的特征向量，本文中采用的是ImageNet网络     
- 将上一步提取的特征向量输入到线性SVM分类器中     

## Region proposals    
本文采用selective search从输入图像中选取候选框。具体方法见[Selective search for object recognition](http://www.huppelen.nl/publications/selectiveSearchDraft.pdf),后续有时间再补充      



## 候选框处理(Object proposal transformations)    
selective search选取的候选框大小不一定能满足ImageNet的输入要求，因此需要对候选框做一些处理，有两种方法    
- 第一种方法用一个最紧凑的框包含待检测目标，然后对矩形框的各个方向同时缩放到需要的尺寸    
- 第二种方法则是将候选框中除目标区域外的部分排除(置灰)    


## 正负样本的选取(Positive vs. negative examples and softmax)    
### IOU的定义    
IOU(intersection-over union)定义两个bounding box的重合度,    
<img src=./pictures/fast_rcnn_4.png>    
$IOU= (A \bigcap B )/(A \bigcup B)$     


在fine tuning阶段，取目标候选框中和ground-true具有最大IoU且IoU至少大于0.5的这些作为正样本，剩下的作为负样本；在训练SVM阶段，只取每个目标类的ground-true作为正样本，Iou小于0.3的作为负样本。

## 边框回归(Bounding-box regression)    
SVM预测出目标的类别后，还需要定位出目标在原始图片中的位置，这一步采用的是边框回归。    
输入是N个训练样本对${(P^i,G^i)}_{i=1,2,...,N}$,$P^i=(P_x^i,P_y^i,P_w^i,P_h^i)$表示候选框中心的像素坐标，下标表示第i个样本；G是对应的ground-true的坐标；现在的目标是学习一个映射，将P映射到G。    
$$
\begin{aligned}
\hat G_x &= P_w d_x(P) + P_x  \text{\quad \quad(1)} \\   
\hat G_y &= P_w d_y(P) + P_y   \text{\quad \quad(2)} \\
\hat G_w &= P_w exp(d_w(P))   \text{\quad \quad(3)} \\
\hat G_h &= P_w exp(d_h(P))   \text{\quad \quad(4)}
\end{aligned}
$$

每个$d_*(P)$都是一个线性函数，输入是候选框P经过ImageNet后第五个池化层的特征向量，用$\phi_5(P)$表示。所以$d_*(P) = w_*^T \phi_5(P)$,$w_*^T$是需要学习的向量参数，$w_*^T$的学习采用的是优化平方误差+二阶正则，   
$$
w_*=arg\,\min_{w_*}  \sum_i^{N} (t_*^i - \hat w_*^T \phi_5(P))^2 + \lambda||w_*||^2
$$
    
回归目标$t_*$定义为：    
$$
\begin{aligned}
t_x &= (G_x - P_x)/P_w  \\
t_y &= (G_y - P_y)/P_h  \\
t_w &= log(G_w/P_w)       \\
t_h &= log(G_h/P_h)   \\
\end{aligned}
$$