---
description: 将向量变为长度为 1 的单位向量，但方向保持不变，可用于计算法向量、摄像机方向等
---

# 归一化 Normalization

假设 v = (3, 4, 5)，使用 `normalize()` 函数可以将其归一化

```swift
let v = SIMD3<Float>(3, 4, 5)
let v_normalized = normalize(v)  // 自动计算归一化
print(v_normalized)  // 输出大约 (0.42, 0.57, 0.71)
```

**法向量计算**：光照计算时，法向量必须归一化，否则光照强度计算会出错 **摄像机方向计算**：摄像机的视线方向 (zAxis) 需要归一化，否则旋转和移动会异常

```
例如：向量 A 指向 xx 方向的，10 个单位长度，则归一化让它长度变成 1 个单位长度，但指向的方向依旧不变
```

> \[!question] 为什么要归一化？ 如果向量的大小不同，它们的计算会受到影响，归一化后，向量长度变成 1，可以专注于方向，而不用考虑大小，归一化向量可以让数值更稳定

**归一化公式**

归一化的向量，通过向量的每个分量除以向量的 [mo-magnitude.md](mo-magnitude.md "mention") 即可计算得出

* **分量**：也就是 x、y、z 这些
* **模**：也就是 |v|

$$
\hat{v} = \frac{v}{|v|}
$$

```
向量 v = (x, y, z)，通过计算得出[[模 Magnitude]]：|v|
```

$$
|v| = \sqrt{x^2 + y^2 + z^2}
$$

```
用原始 x, y, z 分别除以 |v|，得到归一化的 v
```

$$
v_{normalized} = \left( \frac{x}{|v|}, \frac{y}{|v|}, \frac{z}{|v|} \right)
$$

$$
这里的 v_{normalized} 长度就是 1，但它仍然指向原来的方向
$$

**示例**

v = (3, 4, 5)

1. 计算[mo-magnitude.md](mo-magnitude.md "mention") |v|

$$
|v| = \sqrt{3^2 + 4^2 + 5^2} = \sqrt{9 + 16 + 25} = \sqrt{50} \approx 7.07
$$

2. 通过 |v| 归一化

$$
v_{normalized} = \left( \frac{3}{7.07}, \frac{4}{7.07}, \frac{5}{7.07} \right) \approx (0.42, 0.57, 0.71)
$$

```
方向没变，长度变成 1
```
