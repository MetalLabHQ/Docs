# MTLBuffer

现在MTLBuffer 是 **Metal** 中用于在 **CPU 和 GPU 之间共享数据** 的 **内存缓冲区（Buffer）**，通常为**UMA(Unified Memory Architecture 统一内存)**，用于存储 **顶点数据**、**索引数据**、**Uniform 变量**、**计算任务的数据** 等。

* **存储 GPU 可访问的数据**（顶点数据、索引数据、Uniforms、计算任务输入/输出等）
* **提供 Metal 句柄**，GPU 通过 MTLBuffer 访问内存
* **支持不同的存储模式**，优化 CPU-GPU 交互（storageModeShared、storageModePrivate） 等不同 \[\[#MTLResourceOptions 选项]]
* **可用于渲染管线、计算管线和深度学习任务**

在 Buffer 里面的数据会自动进行 \[\[内存布局 Memory Layout#内存对齐 Memory Alignment|内存填充对齐 Memory Padding/Alignment]]，以确保 GPU 能够高效访问数据。**Metal 采用 16 字节（128 bit）对齐规则**，但不同的数据类型和平台可能有额外的对齐要求。下面详细介绍 MTLBuffer 的 **对齐规则、填充原因和如何手动处理对齐**。

**创建 MTLBuffer**

需要从 \[\[MetalKit#创建 MTLDevice|MTLDevice]] 中创建

```swift
let data: [Double] = [1, 2, 3]

// 生成 MTLBuffer
let someBuffer = device.makeBuffer(
    bytes: data,
	length: MemoryLayout<Double>.size * data.count, // Double 长度乘以数组长度
    options: .storageModeShared
)
```

**MTLResourceOptions 选项**

**cpuCacheModeWriteCombined**

* 描述：资源的写操作与 CPU 缓存合并，优化 CPU 和 GPU 对资源的访问。
* 使用场景：适用于 CPU 和 GPU 共享数据的情况，优化 CPU 向 GPU 传输数据的速度，减少同步延迟。
* 额外信息：常用于顶点缓冲区、纹理数据等的共享。

**storageModeShared**

* 描述：资源可以由 CPU 和 GPU 都访问，并且可以在两者之间共享，内容会同步。
* 使用场景：用于 CPU 和 GPU 需要频繁读写的资源，如顶点缓冲区、纹理、常量缓冲区等。
* 额外信息：需要较多的同步操作，性能开销较大。&#x20;

**storageModePrivate**

* 描述：资源仅能由 GPU 访问，CPU 无法直接操作。
* 使用场景：适用于 GPU 专用的资源，如渲染目标、临时纹理、GPU 内部缓存等，CPU 不干预。
* 额外信息：GPU 操作专用，性能最佳。

**storageModeMemoryless**

* 描述：资源没有内存，通常用于渲染目标，GPU 动态生成数据。
* 使用场景：用于帧缓冲、深度缓冲等渲染目标，GPU 每次渲染时生成或更新数据，CPU 不访问。
* 额外信息：渲染目标类资源，内存无需保留。&#x20;

**hazardTrackingModeUntracked**

* 描述：资源不追踪访问状态，不监控资源是否被并发操作。
* 使用场景：用于单线程或没有并发访问的资源，不需要监控访问冲突，避免不必要的性能开销。
* 额外信息：性能优化，适用于没有并发操作的场景。&#x20;

**hazardTrackingModeTracked**

* 描述：资源会自动追踪访问状态，确保没有冲突。
* 使用场景：用于多线程环境，避免 CPU 和 GPU 或多个 GPU 操作之间的资源冲突，自动同步访问。
* 额外信息：适用于复杂渲染管线，多个线程访问相同资源时。
