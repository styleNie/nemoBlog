1.SVM概率输出     
标准的SVM的无阈值输出为    
$$
f(x)=h(x)+b   \tag{1}
$$

其中   
$$
h(x)=\sum_{i}   y_i \alpha_{i} k(x_i,x)   \tag{2}
$$

Platt利用sigmoid-fitting方法，将标准SVM的输出结果进行后处理，转换成后验概率。     
$$
p(y=1|f)=\frac{1}{1+exp(Af+B)}   \tag{3}
$$

A,B为待拟合的参数，f为样本x的无阈值输出。sigmoid-fitting方法的优点在于保持SVM稀疏性的同时，可以良好的估计后验概率。    

2.拟合sigmoid模型 
用极大似然估计来估计公式（3）中的参数A,B。 
定义训练集为$(f_i,t_i)$, $t_i$为目标概率输出值，定义为    
$$
t_i=\frac{y_i +1}{2}
$$

$yi$为样本的所属类别，取值{-1，1} 
极小化训练集上的负对数似然函数     

$$
min -\sum_{i} t_i log(p_i) + (1-t_i)log(1-p_i)  \tag{4}
$$
其中 

$$
p_i=\frac{1}{1+exp(Af_i +B)}
$$
由于sigmoid函数的稀疏性(sigmoid(-5)=0.0067;sigmoid(5)=0.9933)而$t_i$取值{0，1},要完全拟合目标值，就要求sigmoid的输入向实数轴两端靠拢，而sigmoid函数对数轴两端的值变化不敏感，难以区分，所以对$t_i$做一个平滑处理，platt的做法是    

$$
t_i = 
\begin{cases}
\frac{N_+ + 1}{N_+ + 2},  & \text{if $y_i$=+1} \\[2ex]
\frac{1}{N_+ + 2}, & \text{if $y_i$=-1}
\end{cases}  i=1,...,l
$$



其中$N_+$为正样本的数目，$N_-$为负样本的数目。

3.libsvm 
3.1数值问题 
求解（4）时遇到两个数值计算的问题；求解(4)的梯度与hessian矩阵如下     
$$
\nabla F(z)=\left[  \frac{\sum_i f_i(t_i - p_i)}{\sum_i (t_i - p_i)}    \right]
$$

$$
H(z) = \begin{bmatrix} \sum_i f_i^2 p_i(1-p_i) & \sum_i f_i p_i(1-p_i) \\ \sum_i f_i p_i(1-p_i) & \sum_i p_i(1-p_i) \\ \end{bmatrix} 
$$

1) log与exp函数极易溢出，如果$Af_i+B$较大，那么$exp(Af_i+B) →∞$；而当$p_i→0$时，$log(p_i) →∞$,Platt对其作了修正让log(0)返回-200，但并不是一个好的办法    
2) $1-p_i=1- \frac{1}{1+exp(Af_i+B)}$,当$p_i→1$时，极易导致”catastrophic cancellation”问题；如$f_i=1,(A,B)=(-64,0)$，在C++中$1-p_i$将会返回0，而 $\frac{exp(Af_i+B}{1+exp(Af_i+B)}$,则会给出一个精度较好的结果    

3.2 libsvm的改进     
在libsvm中作者对这些问题给出了其解决办法 
将公式（4）变换形式     

$$
\begin{align}
-(t_i log p_i + (1-t_i)log(1-p_i))   \\
&=(t_i -1)(Af_i+B)+log(1+exp(Af_i +B)  \tag {5} \\
&=t_i(Af_i + B)+log(1+exp(-Af_i -B))   \tag {6} \\
\end{align}
$$

具体使用的时候    
If $Af_i+B>0$ use(6) else use (5)   

Reference     
[1] Platt J. Probabilistic outputs for support vector machines and comparisons to regularized likelihood methods[J]. Advances in large margin classifiers, 1999, 10(3): 61-74.     
[2] Lin H T, Lin C J, Weng R C. A note on Platt’s probabilistic outputs for support vector machines[J]. Machine learning, 2007, 68(3): 267-276.

