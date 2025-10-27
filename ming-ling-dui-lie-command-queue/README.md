---
description: GPU 设计之初就是做并发处理，需要一个高效的队列存储一系列要执行的渲染任务，作为 CPU 对 GPU 的桥梁
icon: chart-kanban
---

# 命令队列 Command Queue

#### 创建命令队列

```swift
let commandQueue = device.makeMTL4CommandQueue()
```

CPU 可以提前提交多个任务到队列中，通过 [ming-ling-bian-ma-qi-command-encoder.md](ming-ling-bian-ma-qi-command-encoder.md "mention") 将命令编码成 [ming-ling-huan-chong-qu-command-buffer.md](ming-ling-huan-chong-qu-command-buffer.md "mention")，由于 GPU 是异步执行，CPU 提交完成后无需等待 GPU 完成

<details>

<summary>Q: GPU 设计为了并发，但对命令队列的任务为什么是串行处理？</summary>

A: 命令队列的串行处理是为了保持任务的逻辑顺序（如顶点着色、光栅化、像素着色），避免数据依赖冲突。GPU 内部采用流水线并行架构，使得任务看似串行，但实际上不同硬件单元会并发执行多个阶段。 GPU 从队列中取任务进行分配，分配完成后立马取下一个任务（假设无性能瓶颈），最后提交时按序提交 例如，顶点着色器处理 **A 帧** 时，光线追踪单元可能在计算 **B 帧** 的反射，纹理单元预加载 **C 帧** 数据，处理完成时，按 A→B→C 顺序提交，而不是哪一帧先处理完。

</details>
