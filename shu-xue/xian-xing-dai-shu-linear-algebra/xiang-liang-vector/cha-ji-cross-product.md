---
description: 叉积用于从两个边向量计算垂直于面的法线方向，从而确定光照、面朝向和坐标基的方向。
---

# 叉积 Cross Product

使用 `cross()` 函数对两个向量 A(1, 0, 0) B(0, 1, 0) 进行叉积计算可以得出新的向量 C

```swift
let A = SIMD3<Float>(1, 0, 0)  // X 轴
let B = SIMD3<Float>(0, 1, 0)  // Y 轴
let C = cross(A, B)            // 计算 Z 轴方向
print(C)  // (0, 0, 1)
```

* 用于计算两个向量构成的平面的法线
* 也可以计算右手坐标系中的坐标轴方向
* 可用于 [guan-ce-bian-huan-view-transformation.md](../../../guang-shan-hua-liu-cheng/guan-ce-bian-huan-view-transformation.md "mention")

&#x20;$$\vec{A}\cdot\vec{B}=\vec{C}$$ 示意图：&#x20;

<img src="../../../.gitbook/assets/叉积示意图 Cross Product.svg" alt="" class="gitbook-drawing">

其中 $$|\vec{C}|=|\vec{A}||\vec{B}|\sin(\theta)$$

**叉积 cross(A, B) 会返回一个新的向量 C，这个向量 C：**

* 垂直于 A 和 B 构成的平面
* 方向符合右手法则
* 如果 A 和 B 平行或共线，叉积结果是 (0,0,0)，即没有确定的垂直方向

**计算公式**

$$
A \times B =
\begin{vmatrix}
\mathbf{i} & \mathbf{j} & \mathbf{k} \\
x_1 & y_1 & z_1 \\
x_2 & y_2 & z_2
\end{vmatrix}
$$

展开后：

$$
A \times B =
\Big(
(y_1 z_2 - z_1 y_2),
(z_1 x_2 - x_1 z_2),
(x_1 y_2 - y_1 x_2)
\Big)
$$
