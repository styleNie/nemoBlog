
Unigram和Bigram模板分别生成CRF的状态特征函数$s_l(y_i,x,i)$和转移特征函数$t_k(y_{i-1},y_i,x,i)$。其中$y_i$是标签，$x$是观测序列，$i$是当前节点位置。每个函数还有一个权值，具体请参考CRF相关资料。     

crf++模板定义里的%x[row,col]，即是特征函数的参数$x$。     
举个例子。假设有如下用于分词标注的训练文件：    
```
北    N    B
京    N    E
欢    V    B
迎    V    M
你    N    E
```    

其中第3列是标签，也是测试文件中需要预测的结果，有BME 3种状态。第二列是词性，不是必须的。    
特征模板格式：%x[row,col]。x可取U或B，对应两种类型。方括号里的编号用于标定特征来源，row表示相对当前位置的行，0即是当前行；col对应训练文件中的列。这里只使用第1列（编号0），即文字。   


1、Unigram类型    
每一行模板生成一组状态特征函数，数量是`L*N` 个，L是标签状态数。
N是此行模板在训练集上展开后的唯一样本数，
在这个例子中，第一列的唯一字数是5个，所以有`L*N = 3*5=15`。
例如：U01:%x[0,0]，生成如下15个函数：   
```
func1 = if (output = B and feature=U01:"北") return 1 else return 0
func2 = if (output = M and feature=U01:"北") return 1 else return 0
func3 = if (output = E and feature=U01:"北") return 1 else return 0
func4 = if (output = B and feature=U01:"京") return 1 else return 0
...
func13 = if (output = B and feature=U01:"你") return 1 else return 0
func14 = if (output = M and feature=U01:"你") return 1 else return 0
func15 = if (output = E and feature=U01:"你") return 1 else return 0
```   

这些函数经过训练后，其权值表示函数内文字对应该标签的概率（形象说法，概率和可大于1）。
又如 U02:%x[-1,0]，训练后，该组函数权值反映了句子中上一个字对当前字的标签的影响。

2、Bigram类型     
与Unigram不同的是，Bigram类型模板生成的函数会多一个参数$y_{i-1}$。    
生成函数类似于：    
```
func1 = if (prev_output = B and output = B and feature=B01:"北") return 1 else return 0
```
这样，每行模板则会生成 `L*L*N` 个特征函数。经过训练后，这些函数的权值反映了上一个节点的标签对当前节点的影响。   

每行模版可使用多个位置。例如：U18:%x[1,1]/%x[2,1]字母U后面的01，02是唯一ID，并不限于数字编号。如果不关心上下文，甚至可以不要这个ID。

参考：CRF++: Yet Another CRF toolkit    

Reference: https://www.zhihu.com/question/20279019/answer/273594958