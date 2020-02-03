<script type="text/javascript" async src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML"> </script>

# 1. 向量内积含义
向量： $(a_1,a_2,\cdots,a_n)^T\cdot(b_1,b_2,\cdots,b_n)^T=a_1b_1+a_2b_2+\cdots+a_nb_n$
解释：$A \cdot B=|A||B|cos(a)$

设向量B的模为1，则A与B的内积值等于A向B所在直线投影的矢量长度。

# 2. 基变换

基是一个向量
基是正交的（即内积为0，或直观说相互垂直）
要求：线性无关

变换：数据与第一个基做内积运算，结果作为第一个新的坐标分量，然后与第二个基做内积运算，结果作为第二个新坐标的分量

如坐标点 $(3,2)$ 映射到基
$$
\begin{pmatrix}
   1/\sqrt{2} & 1/\sqrt{2} \\\\
   -1/\sqrt{2} & 1/\sqrt{2}
\end{pmatrix}
$$
中的坐标为：
$$
\begin{pmatrix}
   1/\sqrt{2} & 1/\sqrt{2} \\\\
   -1/\sqrt{2} & 1/\sqrt{2}
\end{pmatrix}
\begin{pmatrix}
   3 \\\\
   2
\end{pmatrix}
=
\begin{pmatrix}
   5/\sqrt{2} \\\\
   -1/\sqrt{2}
\end{pmatrix}
$$

# 3. PCA算法
将一组N维向量降为K维（K大于0，小于N），目标是选择K个单位正交基，使原始数据变换到这组基上，各字段两两间协方差为0，字段的方差则尽可能大。

设有两个特征维度的数据为：
$$
X = \begin{pmatrix}
   a_1 & a_2 & \cdots & a_m \\\\
   b_1 & b_2 & \cdots & b_m
\end{pmatrix}
$$

对应的协方差矩阵：
$$
\frac{1}{m} X X^T = \begin{pmatrix}
   \frac{1}{m}\sum^m_{i=1}a_i^2 & \frac{1}{m}\sum^m_{i=1}a_ib_i \\\\
   \frac{1}{m}\sum^m_{i=1}a_ib_i & \frac{1}{m}\sum^m_{i=1}b_i^2
\end{pmatrix}
$$

矩阵对角线上的两个元素分别是两个字段的方差，而其它元素是a和b的协方差。

实对称矩阵：一个n行n列的实对称矩阵一定可以找到n个单位正交特征向量。证明略。
$$
E = (e_1, e_2, \cdots, e_n)
$$

实对称矩阵进行对角化：
$$
E^TCE=\Lambda=
X = \begin{pmatrix}
   \lambda_1 & 0 & \cdots & 0 \\\\
   0 & \lambda_2 & \cdots & 0 \\\\
    \vdots & & \ddots & \vdots \\\\
   0 & 0 & \cdots & \lambda_n
\end{pmatrix}
$$

根据特征值的从大到小，将特征向量从上到下排列，则用前K行组成的矩阵乘以原始数据矩阵X，就得到了我们需要的降维后的数据矩阵Y。