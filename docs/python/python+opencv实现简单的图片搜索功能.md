原文地址 
http://www.pyimagesearch.com/2014/12/01/complete-guide-building-image-search-engine-python-opencv/

一、图片搜索中的概念解释：    
图片搜索引擎有三种不同的模式    
1.Search by Meta-Data:    元数据搜索模式，这种和传统的文字搜索类似，给索引数据添加文字注释，上传待查询的图片的时候，需要附加图片的文字描述，实际在后台搜索对应的文字描述，典型的有 https://www.flickr.com/    

2.Search by     Example：基于内容的搜索，即Content-Based Image Retrieval (CBIR) systems，也即后文介绍，通过计算图片内容相似度实现搜索，典型的如 https://www.tineye.com/ 

3.前两种的混合

二、分四个步骤实施CBIR图片搜索引擎     
1.定义图片算子：即定义从图片中提特征的函数，原文中使用了色彩直方图作为图片的特征     
2.简历索引数据库：用第一步中定义的提特征函数，遍历图片数据库，提取每张图片的函数，数据保存格式为 image_name ,image_vector     
3.定义相似度函数：即计算从两张图片中提取的特征向量间的距离，原文中用的卡方距离     
4.调用前三步写好的函数，实现一个查询入口    
