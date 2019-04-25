## fast R-CNN分析      
忽略掉候选框的生成部分，fast R-CNN的卷积部分在GPU上的运行速度已经可以做到近实时计算了，所以候选框的生成成了整个系统的瓶颈；卷积部分的速度快是因为利用了GPU，而生成候选框的选择搜索算法依然是在CPU上运行，那能不能将生成候选框的选择搜索算法也在GPU上跑呢？这似乎是个可行的办法，但是再往前进一步，能否用卷积神经网络来替代选择搜索呢。      


## faster R-CNN 结构     
<img src=./pictures/fast_rcnn_2.jpg>       
faster R-CNN分为两个部分，第一个部分为全卷积网络，用于提取候选框；第二个部分为fast R-CNN中的候选框检测部分，即分类与回归。       

### Region Proposal Networks     
原始的输入图像经过卷积和池化层后得到feature map，然后用基于卷积神经网络的区域提名网络(RPN,Region Proposal Networks)从feature map中生成候选框。RPN网络的输入图像可以是任意尺寸，输出矩形的目标候选框及其得分。      




Reference     
1. [Faster RCNN 学习笔记](https://www.cnblogs.com/wangyong/p/8513563.html)      
2. [【目标检测】Faster RCNN算法详解](https://blog.csdn.net/shenxiaolu1984/article/details/51152614)