本文针对R-CNN的缺点做出了针对性的改进(同时分析了SSPnet的优缺点)。    

## R-CNN的缺点     
- 多阶段训练，R-CNN首先用候选框微调ImageNet网络，然后用SVM拟合ImageNet的输出做分类，最后采用边框回归    
- 训练在时间和空间上的消耗较大，从候选框中提取出特征后需要写入硬盘中     
- 检测速度慢，用VGG16网络的检测速度是47s/image(on a GPU)     

## fast R-CNN的改进    
- 单阶段、多目标训练   
- 训练阶段更新网络所有层的参数(针对SSPnet的改进)     
- 不需要将特征缓存到硬盘    

## 训练阶段    
- 输入是224x224的固定大小图片      
- 经过5个卷积层和2个池化层，得到feature map    
- 在feature map上提取候选框(约2000个)，从候选框中提取固定长度的特征向量，这一步称为 ROIPooling层     
- 提取的特征向量分别输入到全连接层，进而到达输出层，分别对应目标分类和回归      
<img src=./pictures/fast_rcnn_1.png>    

## ROIPooling     
由于region proposal的尺度各不相同，而期望提取出来的特征向量维度相同，因此需要某种特殊的技术来做保证。ROIPooling的提出便是为了解决这一问题的。其思路如下：    
- 将region proposal划分为H×W大小的网格    
- 对每一个网格做MaxPooling(即每一个网格对应一个输出值)    
- 将所有输出值组合起来便形成固定大小为H×W的feature map     

## 训练样本
训练过程中每个mini-batch包含2张图像和128个region proposal（即ROI，64个ROI/张），其中大约25%的ROI和ground truth的IOU值大于0.5（即正样本），且只通过随机水平翻转进行数据增强。   
