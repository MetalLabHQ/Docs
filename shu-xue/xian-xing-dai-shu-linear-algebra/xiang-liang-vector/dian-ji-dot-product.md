---
description: 通过两个向量点乘，找到向量之间的夹角，可以用来计算向量之间的相似性、投影、夹角度数等信息，也可以判断一个点是否在三角形内
---

# 点积 Dot Product

通过两个向量 A(1, 2, 3) B(4, 5, 6) 进行点乘计算 32

```swift
let A = SIMD3<Float>(1, 2, 3)
let B = SIMD3<Float>(4, 5, 6)
let result = dot(A, B)  // A ⋅ B 计算 1×4 + 2×5 + 3×6 = 32
```

* 当两个向量都是 [dan-wei-xiang-liang-unit-vector.md](dan-wei-xiang-liang-unit-vector.md "mention") 时，非常好计算
* 点积也被称为 Scalar Product
* 找到两个向量之间的夹角
* 也可以用于计算一个向量投影到另一个向量上是什么样的
* 可以对一个向量进行垂直、水平等方向的分解
* 用于计算两个向量是否接近

可以被表示为 $$\vec{a} \cdot \vec{b} = |\vec{a}| |\vec{b}| \cos(\theta)$$ 即 AB 向量的长度 乘 $$\cos(\theta)$$ ，也等同于： $$\cos(\theta) = \frac{\vec{a} \cdot \vec{b}}{|\vec{a}||\vec{b}|}$$ 对于 [dan-wei-xiang-liang-unit-vector.md](dan-wei-xiang-liang-unit-vector.md "mention") 而言，点乘运算将变得简单： $$\cos(\theta) = \hat{a} \cdot \hat{b}$$

两个 3D 向量 A(Ax, Ay, Az) 和 B(Bx, By, Bz) 的点积计算公式如下，其他维度同理

$$
\vec{A} \cdot \vec{B} = 
\begin{bmatrix}
A_x \\
A_y \\
A_z
\end{bmatrix}
\cdot
\begin{bmatrix}
B_x \\
B_y \\
B_z
\end{bmatrix}
= A_x B_x + A_y B_y + A_z B_z
$$

或者用向量符号表示：

$$
A \cdot B = \sum_{i=1}^{n} A_i B_i
$$

也可以计算 AB 向量的长度乘向量夹角的余弦

#### **计算两个向量的夹角**

* |A| 和 |B| 是向量的 [mo-magnitude.md](mo-magnitude.md "mention")
* θ 是两个向量之间的夹角。
* cos(θ) 反映了向量之间的相似性。

$$
\cos(\theta) = \frac{A \cdot B}{|A| |B|}
$$

**这意味着：**

* 如果 dot(A, B) > 0：两个向量的方向相似（夹角小于 90°）
* 如果 dot(A, B) = 0：两个向量正交/垂直（夹角 = 90°）
* 如果 dot(A, B) < 0：两个向量的方向相反（夹角大于 90°）
