---
description: GPU 逐条读取命令缓冲区中的指令并执行
---

# 命令缓冲区 Command Buffer

[.](./ "mention") 存储多个命令缓冲区，并按顺序提交给 GPU，这些命令缓冲区由 [ming-ling-bian-ma-qi-command-encoder.md](ming-ling-bian-ma-qi-command-encoder.md "mention") 生成

**创建命令缓冲区**

使用 `makeCommandBuffer()` 从 [.](./ "mention") 中创建 **命令缓冲区 Command Buffer**

```swift
commandBuffer = commandQueue.makeCommandBuffer()
```

<details>

<summary>Q: 看似一个缓冲，为何叫缓冲区？</summary>

A: 他是一整个数据结构的容器，可以看作一个任务包，包含了对 GPU 的各种指令

</details>

