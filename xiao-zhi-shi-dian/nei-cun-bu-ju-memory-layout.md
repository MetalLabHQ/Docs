# 内存布局 Memory Layout

```swift
MemoryLayout<T>.size/stride/alignment
```

#### size

表示类型 T 实际占用的数据大小（以字节为单位），即该类型实例在未对齐状态下的“裸数据”大小，它不含由内存对齐而引入的填充字节，仅反映其成员字段所占的原始字节数。

* 类型 T 的实际大小，也可以是存储该类型需要的最小字节数
* 不包含额外对齐填充
* 精确反映了占用大小，没有考虑对齐
* 在数组或者缓冲中，通常会因为对齐规则占用额外空间，但是 size 只关注单个值的实际大小，但依旧慎用，只有在动态分配内存时知道确切大小时才建议使用

#### stride

以满足对齐需求。

用于计算数组中元素之间的间隔，表示类型 T 在内存中实际占用的字节数，相当于**实际占用内存字节数**，代表顶点数据之间的内存间隔，因此，**stride 总是 ≥ size**，并且常用于计算数组总长度、指针运算等场景。

* 表示类型 T 在内存中实际占用的总字节，包含了
* 在实际的内存布局中，类型可能需要对齐到某个边界，而 stride 反映了这种实际占用的大小
* 类型在内存中的排列间隔由 stride 决定，stride 总是大于或等于 size，因为它是类型在内存中占用的完整大小

#### alignment

表示类型 T 的内存对齐边界，即该类型在内存布局中对其起始地址的要求（地址必须是 alignment 的整数倍）。

* alignment 表示一个类型在内存中对齐的要求，比如 alignment 为 4 时，一个 3 字节的数据会被填充至 4 字节
* CPU 访问内存时的优化规则，要求类型的起始地址是 alignment 的倍数
* alignment 是类型所有字段中最大的对齐要求

**Example**

```swift
struct Example {
    var a: UInt8   // 占 1 字节
    var b: UInt16  // 占 2 字节
}
```

将其输出

```swift
print(MemoryLayout<Example>.size)      // 3，因为实际占用 3，size 不会计算填充
print(MemoryLayout<Example>.stride)    // 4，因为 a 加了 1 字节的填充
print(MemoryLayout<Example>.alignment) // 2，这是对齐的边界，决定了 a 需要填充 1 字节
```

#### 内存对齐 Memory Alignment

内存对齐是计算机硬件对数据存储的优化规则，目的是让数据在内存中的排列能够高效地被处理器访问

计算机的内存是按**字节**来划分的。每个字节都有一个地址，未对齐的数据会导致额外的性能开销，甚至在某些平台上会导致程序崩溃。

* 某些类型的数据必须存储在特定的地址上，这叫 **对齐边界**，UInt16 类型（占 2 字节）必须存储在 **能被 2 整除的地址上**
* CPU 读取内存时，通常一次会读取 2 字节、4 字节、8 字节甚至更多。如果数据没有对齐到这些边界，CPU 可能需要多次访问内存，效率会下降
* 某些 CPU 或硬件设备可能不支持非对齐的数据访问。如果数据没有对齐，程序可能会崩溃
* 如 GPU 一次拿 16 字节数据，而内存中时 1-12 为第一个数据，13-27 是第二个数据，那么这样会导致崩溃

#### 对齐填充 Alignment Padding

对齐填充是指为了满足 [#nei-cun-dui-qi-memory-alignment](nei-cun-bu-ju-memory-layout.md#nei-cun-dui-qi-memory-alignment "mention") 的规则，在数据之间添加的额外字节。对齐填充不会存储实际数据，但它确保数据在内存中的起始地址符合对齐要求，从而提高 CPU 的访问效率。 **优化内存布局方案与适用范围：**

对于 Metal 而言切勿轻易使用优化布局 在 Metal 中，如果将两个属性存储在一个固定的内存块内如 16字节，而 GPU 期望一次性读取完整的 16 字节作为一个向量，那么该行为会导致崩溃或数据异常，所以 Metal 使用该方式会非常危险，最好的方式还是使用 [simd-shu-ju-jie-gou.md](simd-shu-ju-jie-gou.md "mention")

```swift
struct Example {
    var a: UInt8    // 占 1 字节，实际填充 3 字节
    var b: UInt32   // 占 4 字节，不填充
    var c: UInt8    // 占 1 字节，填充 3 字节
}
```

Example 的 alignment 理应是4字节，所以输出`MemoryLayout<Example>.stride`为12 但将其优化顺序，将最大的属性排至最前，使填充只发生一次

```swift
struct Example {
    var b: UInt32   // 占 4 字节
    var a: UInt8    // 占 1 字节
    var c: UInt8    // 占 1 字节
}
```

此时，内存排布到 6 字节时，只需要填充 2 字节，此时输出 `MemoryLayout<Example>.stride`为8

**适用范围**：

* 嵌入式设备、移动设备或实时系统，内存资源高度受限场景
* 需要动态分配或频繁传递、网络带宽受限场景 但无论如何，优化布局都是非常危险的行为，非必要请勿轻易尝试。
