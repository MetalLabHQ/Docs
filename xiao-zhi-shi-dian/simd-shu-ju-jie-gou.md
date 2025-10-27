---
description: 并行计算！高性能计算！边缘计算！
---

# SIMD 数据结构

**SIMD** 的全称是 **Single Instruction, Multiple Data**，中文称为“单指令多数据流”。这是一个并行计算技术，允许单个指令同时操作多个数据。SIMD 的主要特点是可以极大地提升对数组或矩阵等大量数据进行相同运算的效率，比如在图形学、物理计算和机器学习中的应用。

在 Metal 中，SIMD 是专门用于表示 向量 Vector 的数据类型，例如二维、三维和四维向量。SIMD 数据结构利用硬件支持的向量运算功能，在 CPU 或 GPU 上执行更高效的计算，为了保证不受 \[\[内存布局 Memory Layout#对齐填充 Alignment Padding|优化填充]] 的影响，SIMD 都会将内存填充至 16 字节
