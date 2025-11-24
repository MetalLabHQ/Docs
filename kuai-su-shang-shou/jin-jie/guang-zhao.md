---
description: 神说，要有光
---

# 光照

## Lambert 余弦定律

想想你手上拿着一个手电筒，照射于墙面

<details>

<summary>如果你有 iPhone 的话</summary>

开启 iPhone 手电筒后，像我这样设置：

<figure><img src="../../.gitbook/assets/ iPhone 手电筒.jpeg" alt="" width="129"><figcaption><p>聚光灯的 iPhone 手电筒</p></figcaption></figure>

</details>

**将光束垂直照射在墙面上：**

光斑是一个圆形，而由于所有光的能量都集中在这个圆圈中，所以这里非常亮！

**慢慢的倾斜手电筒：**

光斑变成了一个拉长的椭圆，总面积更大了，但似乎没那么亮了

从这里可以看出，光的总能量没变，但光线越倾斜，**单位面积**收到的能量就越少，看起来就越暗。

#### 基于角度的解读

通过 [#yong-normal-shang-se](../metal-sao-mang/jia-zai-mo-xing.md#yong-normal-shang-se "mention")中可以了解到，当时有一个 Normal 法线没有解释清楚，**法线（Normal）** 用来描述平面朝向的向量。

从上述现象可以得出，对于墙壁的单位面积光照强度与照射角度的关系是：

$$
I = I_{0} \cdot \cos(\theta)
$$

* $$I$$ 代表 Intensity，表面实际接收到的光照强度。
* $$I_0$$ 代表光源垂直照射时的最大强度，即光源本身的强度。
* $$\theta$$代表入射光线与法线之间的夹角。

#### 从图形学角度解读

在图形学中，计算三角函数性能稍微高了一些，使用 点积 Dot Product 的几何性质区代替，即：

$$
Color_{diffuse} = K_d \cdot I_{light} \cdot \max(0, \vec{N} \cdot \vec{L})
$$

这个公式为：

