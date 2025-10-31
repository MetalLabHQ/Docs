---
description: Hello, Metal.
---

# 关于 Metal Lab

### Metal Lab 是什么？

**Metal Lab 是基于 Apple Metal API 生态的中文图形学社区**，致力于推进 Metal 图形学生态，为初学者提供一个学习的方向

Metal Lab 上大部分内容引用自 [Apple Metal 文档](https://developer.apple.com/documentation/metal)、Games 101、RTR4 等教程。

### 为什么需要 Metal Lab 中文文档？

在中文互联网上，关于 **Metal 的教程**资源非常稀少，尽管 Apple 官方提供了详尽的文档和示例代码，但这些文档对于国内开发者而言往往**语言晦涩、结构复杂，缺乏通俗易懂的入门指导**，初学者常常感到难以上手。

此外，**中文社区对 Metal 的关注度相对较低**，很多开发者仍然习惯使用 OpenGL 或 Unity、Cocos、Unreal 之类的引擎来开发图形应用，缺乏对 Metal 本地开发的深入了解。

因此，我们的教程文档旨在填补这一空白，为中文开发者提供一份**系统性、易懂、循序渐进的 Metal 入门资料**，帮助他们从零开始掌握 Metal 编程的精髓。

### 为什么要学习 Metal？

你可能会问：“为什么我要学习 Metal 而不是 OpenGL 或 Vulkan？” 其实，**Metal 是苹果公司专门为自己的硬件平台设计的图形和计算 API**，它不仅为开发者提供了更直接的硬件控制能力，还巧妙地在底层与高层之间找到了一个很好的平衡点。

和 OpenGL 相比，Metal 的渲染命令更加直观、简洁，**它让你能以更低的抽象层级接触 GPU**，从而获得更高的性能和更精细的控制。而与 Vulkan 相比，Metal 又没有那种“过度封装”的感觉，虽然它比 Vulkan 稍高一层，但依然保持着极高的灵活性和效率。

换句话说，**Metal 是一个「适中」的 API**：它不像 OpenGL 那样封装太多，导致性能损耗和控制受限；也不像 Vulkan 那样需要你处理太多底层细节，造成学习成本飙升。它为你提供了一个清晰、简单且高性能的接口，让开发者能够真正“贴近硬件”，而不被繁杂的代码结构所拖累。

尤其是对于希望在 iOS、macOS、visionOS 平台开发高性能图形应用的开发者而言，Metal 是唯一被 Apple 官方推荐的图形 API，具有广泛的应用前景和更长远的技术生命周期。

随着 AI 时代来临，**MPS (Metal Performance Shaders)** 让 GPU 运行卷积、矩阵乘法、激活函数 AI 模型的核心算子。相比自己手写 Metal shader，MPS 已经针对 Apple Silicon 进行了深度优化，能够自动利用 SIMD 指令、内存层次结构等硬件特性。
