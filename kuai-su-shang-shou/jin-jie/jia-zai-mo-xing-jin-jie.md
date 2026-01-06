---
description: 后面还会更进一步
---

# 加载模型进阶

在之前的课程中，我们学习了如何加载 OBJ 模型，但如果你尝试过使用自己的模型去加载的话，往往会报错或加载出错

同时 OBJ 也是一个很基础的 3D 模型格式，很难被

&#x20;

下载该工程作为起点：

{% file src="../../.gitbook/assets/LoadUSD.7z" %}
初始工程
{% endfile %}

可以下载下方的 USDZ 模型并放进 Assets 路径：

{% file src="../../.gitbook/assets/Wooden.usdz" %}

随后你可能会尝试在 AssetsLoader 中，Load 这个模型

```swift
let asset = AssetsLoader.loadAssets(named: "Wooden", ext: "usdz", device: device)!
```

但很快我们就遇到了第一个问题：

```swift
这一行：
let vertexDescriptor = MTKMetalVertexDescriptorFromModelIO(asset.vertexDescriptor!)!
报错了：
// Unexpectedly found nil while unwrapping an Optional value
```

这是因为不同于 OBJ，单个 USDZ 文件内可能会有多种顶点布局方式，所以此处 ModelIO 并没有自作聪明去推断它的顶点布局，需要我们手动去定义顶点布局

#### 自定义默认布局

在 App 路径下创建 VertexLayouts 并定义一套我们自己的默认顶点布局，这里我们只需要关注顶点与法线就好（按需加载）：

{% code title="VertexLayouts.swift" expandable="true" %}
```swift
import MetalKit

extension MTLVertexDescriptor {
    static var `defaultLayout`: MTLVertexDescriptor {
        let mtlVertexDescriptor = MTLVertexDescriptor()

        var offset = 0
        mtlVertexDescriptor.attributes[0].format = .float3
        mtlVertexDescriptor.attributes[0].offset = offset
        mtlVertexDescriptor.attributes[0].bufferIndex = 0
        offset += MemoryLayout<SIMD3<Float>>.stride

        mtlVertexDescriptor.attributes[1].format = .float3
        mtlVertexDescriptor.attributes[1].offset = offset
        mtlVertexDescriptor.attributes[1].bufferIndex = 0
        offset += MemoryLayout<SIMD3<Float>>.stride
        
        mtlVertexDescriptor.layouts[0].stride = offset

        return mtlVertexDescriptor
    }
}

extension MDLVertexDescriptor {
    static var `defaultLayout`: MDLVertexDescriptor {
        let mdlVertexDescriptor = MTKModelIOVertexDescriptorFromMetal(.defaultLayout)
        
        (mdlVertexDescriptor.attributes[0] as? MDLVertexAttribute)?.name = MDLVertexAttributePosition
        (mdlVertexDescriptor.attributes[1] as? MDLVertexAttribute)?.name = MDLVertexAttributeNormal

        return mdlVertexDescriptor
    }
}
```
{% endcode %}

并且在 AssetsLoader 中，使用我们刚定义的默认布局，修改 loadAssets 函数：

<pre class="language-swift" data-title="AssetsLoader.swift"><code class="lang-swift">static func loadAssets(named filename: String, ext: String, device: MTLDevice) -> MDLAsset? {
    guard let url = Bundle.main.url(forResource: filename, withExtension: ext) else {
        print("Cannot find file: \(filename).\(ext)")
        return nil
    }
    
    if !MDLAsset.canImportFileExtension(ext) {
        print("File extension .\(ext) is not supported for import")
    }
    
    let asset = MDLAsset(
        url: url,
<strong>        vertexDescriptor: .defaultLayout,
</strong>        bufferAllocator: MTKMeshBufferAllocator(device: device)
    )
    
    return asset
}
</code></pre>

运行工程，你应该能看到一个灰色的盒子旋转

结束了么？让我们换一个 USDZ 模型试试：

{% file src="../../.gitbook/assets/Hammer (1).usdz" %}

{% code title="Renderer.swift" %}
```swift
let camera = Camera(
    position: SIMD3<Float>(0, 0, 15), // 调整了一下相机位置
    target: SIMD3<Float>(0, 0, 0),
    up: SIMD3<Float>(0, 1, 0)
)

let asset = AssetsLoader.loadAssets(named: "Wooden", ext: "usdz", device: device)!
```
{% endcode %}

当你把 asset 替换为 Hammer.usdz 时你会发现，整个模型完全加载不出来，这是因为这个模型拥有不止一个 Submesh，对于复杂的模型而言，需要在规范化且不会被回收的，保证在 Draw 的时候 GPU 还能访问到这些复杂的数据，这就要引出我们的 MTLResidencySet 显式 GPU 资源驻留机制

#### MTLResidencySet

MTLResidencySet 是苹果在 iOS 18+ 中引入的显式 GPU 资源管理模型，用于显式声明哪些资源（缓冲区、纹理、堆）在这个 Set（集合）内，对 GPU 可访问

以往只能依靠 `useResource` / `useHeap` 等 API，ResidencySet 是批量聚合的，只在 `commit()` 时生效

并且 ResidencySet 绑定到 **MTL4CommandBuffer** 或 **MTL4CommandQueue** 中，里面所有资源对该命令缓冲下的所有编码器都可用，不用反复声明

接下来，在 Renderer 中声明 ResidencySet：

{% tabs %}
{% tab title="创建 MTLResidencySet" %}
{% code title="Renderer.swift" %}
```swift
let residencySet: MTLResidencySet

init() {
    // MARK: - State
    // 持久化资源
    self.residencySet = try device
        .makeResidencySet(descriptor: MTLResidencySetDescriptor())
    
    // MARK: - Command Queue
    // 绑定到命令队列上
    self.commandQueue.addResidencySet(residencySet)
}
```
{% endcode %}
{% endtab %}

{% tab title="init() 完整代码" %}
<pre class="language-swift" data-title="Renderer.swift"><code class="lang-swift">let residencySet: MTLResidencySet

init(device: MTLDevice) throws {
    self.device = device
    
    let asset = AssetsLoader.loadAssets(named: "Hammer", ext: "usdz", device: device)!
    AssetsLoader.printAssetInfo(asset: asset)
    
    for i in 0..&#x3C;asset.count {
        let object = asset.object(at: i)
        entities.append(Entity(object: object, device: device))
    }
    
    let vertexDescriptor = MTKMetalVertexDescriptorFromModelIO(asset.vertexDescriptor!)!
    
    // MARK: - Buffers
    self.uniformsBuffer = device.makeBuffer(
        length: MemoryLayout&#x3C;Uniforms>.size
    )!
    
    // 参数表
    let argTableDescriptor = MTL4ArgumentTableDescriptor()
    argTableDescriptor.maxBufferBindCount = 2
    self.argumentTable = try device.makeArgumentTable(descriptor: argTableDescriptor)
    self.argumentTable.setAddress(uniformsBuffer.gpuAddress, index: 1)
    
    
    // MARK: - Load Shaders
    let library = device.makeDefaultLibrary()!
    
    // 顶点着色器
    let vertexFunctionDescriptor       = MTL4LibraryFunctionDescriptor()
    vertexFunctionDescriptor.library   = library
    vertexFunctionDescriptor.name      = "vertex_main"
    
    // 片元着色器
    let fragmentFunctionDescriptor     = MTL4LibraryFunctionDescriptor()
    fragmentFunctionDescriptor.library = library
    fragmentFunctionDescriptor.name    = "fragment_main"
    
    
    // MARK: - Descriptor
    // 渲染管线描述符
    let pipelineDescriptor = MTL4RenderPipelineDescriptor()
    pipelineDescriptor.vertexFunctionDescriptor        = vertexFunctionDescriptor
    pipelineDescriptor.fragmentFunctionDescriptor      = fragmentFunctionDescriptor
    pipelineDescriptor.vertexDescriptor                = vertexDescriptor
    pipelineDescriptor.colorAttachments[0].pixelFormat = .bgra8Unorm
    
    // 深度模板描述符
    let depthStateDescriptor = MTLDepthStencilDescriptor()
    depthStateDescriptor.depthCompareFunction = .less
    depthStateDescriptor.isDepthWriteEnabled = true
    
    // MARK: - State
    // 渲染管线状态
    self.pipelineState = try device
        .makeCompiler(descriptor: MTL4CompilerDescriptor())
        .makeRenderPipelineState(descriptor: pipelineDescriptor)
    // 深度测试状态
    self.depthState = device
        .makeDepthStencilState(descriptor: depthStateDescriptor)!
<strong>    // 创建持久化资源
</strong><strong>    self.residencySet = try device
</strong><strong>        .makeResidencySet(descriptor: MTLResidencySetDescriptor())
</strong>    
    
    // MARK: - Command Queue
    self.commandQueue = device.makeMTL4CommandQueue()!
    self.commandBuffer = device.makeCommandBuffer()!
    self.commandAllocator = device.makeCommandAllocator()!
    // 绑定到命令队列上
<strong>    self.commandQueue.addResidencySet(residencySet)
</strong>    
    super.init()
}
</code></pre>
{% endtab %}
{% endtabs %}

就完成了 ResidencySet 的创建，接下来就是将数据写进 Set 中

#### MTLAllocation

MTLAllocation 代表 GPU 显存中的一次实际分配，Buffer、Texture 等资源都遵循了 MTLAllocation 协议，可以把 UniformsBuffer 和模型所需要的 VertexBuffer 与 IndexBuffer 都放进 `allocations` 中

由于本身就是一次分配显存的行为，所以可以在初始化的 init 中创建：

{% tabs %}
{% tab title="创建 allocations" %}
{% code title="Renderer.swift" %}
```swift
init() {
    // MARK: - Residency Allocation Collection
    var allocations: [MTLAllocation] = []
    allocations.append(uniformsBuffer)
    
    for entity in entities {
        for mesh in entity.meshes {
            allocations.append(contentsOf: mesh.vertexBuffers)
            for submesh in mesh.submeshes {
                allocations.append(submesh.indexBuffer)
            }
        }
    }
}
```
{% endcode %}
{% endtab %}

{% tab title="如果你想炫技的话..." %}
稍等，在使用下面这段代码之前，不妨思考一下，一周后还能不能一眼看懂？

```swift
let allocations: [MTLAllocation] = [uniformsBuffer] + entities
    .flatMap { $0.meshes }
    .flatMap { mesh in
        mesh.vertexBuffers + mesh.submeshes.map { $0.indexBuffer }
    }
```
{% endtab %}
{% endtabs %}

最后一次性提交到 ResidencySet 上，这样 GPU 就能访问到了：

{% code title="Renderer.swift" %}
```swift
// MARK: - State
// 创建持久化资源
self.residencySet = try device
    .makeResidencySet(descriptor: MTLResidencySetDescriptor())
self.residencySet.addAllocations(allocations)
self.residencySet.commit()
```
{% endcode %}

至此再运行工程，你就会看到这个 Hammer.usdz 了

<figure><img src="../../.gitbook/assets/没有累计 TransformStack 的 USDZ.png" alt="" width="375"><figcaption></figcaption></figure>

Hammer 有了，但这 Hammer 为什么不像 Hammer？

#### Reality Composer Pro

Reality Composer Pro 可帮助你导入和整理 3D 模型、材质和声音等素材，使用 Reality Composer Pro 可以看到一个模型的详细信息，免去了下载 Blender、Maya 等专业应用的步骤

> PS：对 Mac 触控板操作非常友好，且界面没有那种 3D 软件的登味

在 Xcode 菜单中，进入 Xcode -> Open Developer Tool -> Reality Composer Pro

<figure><img src="../../.gitbook/assets/Reality Composer Pro 的入口.png" alt="" width="375"><figcaption><p>Reality Composer Pro 的入口在 Xcode 的菜单</p></figcaption></figure>

创建一个简单的工程，然后把 Hammer.usdz 拖进去，再放在左侧的导航栏中：

<figure><img src="../../.gitbook/assets/Hammer.usdz 的结构.png" alt=""><figcaption></figcaption></figure>

点击导航栏中的 Hammer，会发现右侧检查器的 Transform 中，它们有三个属性：Position、Rotation、Scale，如果你拥有一些线性代数基础的话，你就会知道这里的三个属性，最终可以用一个 4×4 Transform Matrix，这就是它真正的 Model Matrix(模型矩阵)

展开发现有两个子节点叫 Circle 和 Cylinder，依次点击它们，从检查器中可以看到它们每个节点中，Transform 的值是不一样的，如果你尝试给 position、rotation 都设置为 0 的话，会得到：

<figure><img src="../../.gitbook/assets/Position 和 Rotation 都为 0 的 Hammer.png" alt="" width="375"><figcaption></figcaption></figure>

至此，你应该知道是怎么一回事了，我们在处理 Hammer 模型的时候，并没有去考虑它们的 Transfom，那按照我们在代码中对 Model Matrix 的定义：

```swift
let modelMatrix = float4x4(rotationY: timer)
```

我们就是拿了一个单位矩阵，配上一个根据时间绕 Y 轴旋转的 rotation，是时候使用它本身的 Transform 去构成正确的 Model Matrix 了

