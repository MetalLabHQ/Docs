---
description: 有长度，有方向的值，表示方向和大小的数学对象，从一点指向另一点的箭头，也可以描述 2D 3D 场景中的位置变化等等用途，是非常基础的元素
---

# 向量 Vector

图形学中，使用 [simd-shu-ju-jie-gou.md](../../../xiao-zhi-shi-dian/simd-shu-ju-jie-gou.md "mention") 表示矩阵，此处就可以作为从原点指向 (1, 2) 的向量

```swift
let vectorA = SIMD2<Float>(1, 2)
```

向量可以随意平移，因为只要指向同一个方向，就是同一个向量

* 不关心绝对的开始位置
* 向量的长度被称为向量的 [mo-magnitude.md](mo-magnitude.md "mention")
* [dan-wei-xiang-liang-unit-vector.md](dan-wei-xiang-liang-unit-vector.md "mention") 通过向量的 [mo-magnitude.md](mo-magnitude.md "mention") 除以长度，也就是 [gui-yi-hua-normalization.md](gui-yi-hua-normalization.md "mention") 计算而来
* 始终认为向量是从原点出发
* 向量有两种乘法，分别是 \[\[点积 Dot Product]] 与 \[\[叉积 Cross Product]]

**向量的写法**

在图形学中，向量的写法为列向量，也就是说，如果一个三维向量，那他的写法是 $$A=\begin{bmatrix}X \\ Y \\ Z\end{bmatrix}$$&#x20;

通常写作： $$\vec{a}$$ 或者 $$\vec{AB}$$，将其认为是从原点 A 出发，指向 B

<figure><img src="../../../.gitbook/assets/二维向量示意图 Vector.png" alt=""><figcaption></figcaption></figure>

也可以看作 B - A 的值：$\vec{AB} = B - A$

**向量相加减**

* **平行四边形法则**：两个向量相加，以它们为邻边构成平行四边形，对角线就是和向量
* **三角形法则**：将第二个向量的起点移到第一个向量的终点，从第一个起点到最后一个终点的向量就是和向量

假设有向量 $$\vec{A} = \begin{bmatrix}1 \ 2\end{bmatrix}$$ 与 $$\vec{B} = \begin{bmatrix} 3 \ 4\end{bmatrix}$$

那么 向量相加：

$$
\vec{A} + \vec{B} = \begin{bmatrix}
1 + 3 \\ 2 + 4
\end{bmatrix} = \begin{bmatrix}
4 \\ 6
\end{bmatrix}
$$

向量相减

$$
\vec{A} - \vec{B} = \begin{bmatrix}
1 - 3 \\
2 - 4
\end{bmatrix} = \begin{bmatrix}
-2 \\ -2
\end{bmatrix}
$$

向量相乘见 \[\[线性变换 Linear Transformation]]
