<!-- 
---
title: "Tensorflow中的卷积模块"       
author:        
date: April 16, 2018        

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
# <center>Tensorflow中的卷积模块</center>  

参考 `tensorflow\python\ops\gen_nn_ops.py`中的`conv2d`函数的说明   

输入x格式：`[batch, in_height, in_width, in_channels]`   
输入filter:`[filter_height, filter_width, in_channels, out_channels]`   

```python
output[b, i, j, k] =
          sum_{di, dj, q} input[b, strides[1] * i + di, strides[2] * j + dj, q] *
                          filter[di, dj, q, k]
```

`padding:SAME`  输出filter和输入filter的height,width一致，不足部分补零   
`padding:VALID` 只取有效部分   


官方代码 mnist_deep.py为例   
```html
[-1, 28, 28, 1]        输入
[5,  5,  1,  32]       卷积,SAME
[-1, 28, 28, 32]       第一个卷积输出
[-1, 14, 14, 32]       2*2 SAME 降采样输出
[5,  5,  32, 64]       卷积，SAME
[-1, 14, 14, 64]       第二个卷积输出
[-1, 7, 7,   64]       2*2 SAME 降采样输出  
```

 
