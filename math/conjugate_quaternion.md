# 共轭四元数

设四元数为

\[
q=a+bi+cj+dk,\quad a,b,c,d\in\mathbb R
\]

其中

\[
i^2=j^2=k^2=ijk=-1
\]

它的**共轭四元数**定义为把虚部 \(i,j,k\) 的系数都变号：

\[
\bar q=q^*=a-bi-cj-dk
\]

若把 \(q\) 写成标量部分加向量部分：

\[
q=a+\mathbf v,\quad \mathbf v=bi+cj+dk
\]

则

\[
\bar q=a-\mathbf v
\]

## 例子

例如：

\[
q=1+2i+3j+4k
\]

则

\[
\bar q=1-2i-3j-4k
\]

## 主要性质

1. 双重共轭还原：

\[
\overline{\bar q}=q
\]

2. 加法共轭：

\[
\overline{q+r}=\bar q+\bar r
\]

3. 乘法共轭要反序：

\[
\overline{qr}=\bar r\,\bar q
\]

这是四元数非交换性的体现。

4. 与范数的关系：

\[
q\bar q=\bar q q=a^2+b^2+c^2+d^2=|q|^2
\]

5. 若 \(q\neq 0\)，则逆为：

\[
q^{-1}=\frac{\bar q}{|q|^2}
\]

特别地，若 \(q\) 是单位四元数，即 \(|q|=1\)，则

\[
q^{-1}=\bar q
\]

6. 若 \(q\) 是纯虚四元数，即 \(a=0\)，则

\[
\bar q=-q
\]

## 几何意义

单位四元数

\[
q=\cos\frac{\theta}{2}+\mathbf u\sin\frac{\theta}{2}
\]

表示绕单位轴 \(\mathbf u\) 旋转角 \(\theta\)。

其共轭

\[
\bar q=\cos\frac{\theta}{2}-\mathbf u\sin\frac{\theta}{2}
\]

表示方向相反的旋转。

## 总结

四元数的共轭就是：

- 保持实部不变；
- 虚部整体取负；
- 类似复数共轭；
- 但乘法共轭会反转顺序。

即：

\[
\overline{qr}=\bar r\,\bar q
\]