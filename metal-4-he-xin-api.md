# Metal 4 核心 API

**Metal 4** 是 Apple 推出的**最新一代图形和计算 API**，相较于之前的版本，它在性能、功能和易用性方面都有显著提升，它专为 A15 仿生芯片及后续的 A 系列和 M 系列芯片优化，带来了更高效的渲染管线、更精细的 GPU 控制以及更强的计算能力。相比 Metal 3，Metal 4 支持更复杂的着色语言扩展，提升了图像处理和光照效果的质量，并引入了更智能的资源管理机制，帮助开发者减少内存开销和延迟，从而实现更流畅的图形渲染，因此本教程也从最新一代的 Metal 4 入手。

* [x] TODO 这部分应当在后续用到的时候同步介绍，不能提前介绍晦涩的概念

Metal 4 为现有概念引入了新类型：

* Command queues
* Command buffers
* Command encoders
* Command allocators
* Argument tables
* Texture view pools
* Next generation barriers
