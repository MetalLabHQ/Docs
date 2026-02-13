---
description: Wooden见见 USD 模型格式
---

# 加载多 Mesh 模型

在之前的课程中，我们学习了如何加载 OBJ 模型，但如果你尝试过使用自己的模型去加载的话，往往会报错或加载出错

同时 OBJ 也是一个很基础的 3D 模型格式，不支持很多高级特性，我们需要使用更加现代化的模型格式：**USD: Universal Scene Description**，它最早由皮克斯提出，现已被影视行业广泛采用。

对于游戏行业而言，虽然 FBX 仍是主流，但也有越来越多的公司开始采用 USD 格式作为不同美术应用的中间资产格式，而 Remedy 这样的技术流已经广泛使用 USD 格式了。

下载该工程作为起点：

{% file src="../../.gitbook/assets/LoadUSD.7z" %}
初始工程
{% endfile %}

完整代码：

{% embed url="https://github.com/MetalLabHQ/LoadUSD.git" %}

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

在 App 路径下创建 VertexLayouts.swift 并定义一套我们自己的默认顶点布局，这里我们只需要关注顶点与法线就好（按需加载）：

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

记得删掉之前的 ：

```swift
let vertexDescriptor = MTKMetalVertexDescriptorFromModelIO(asset.vertexDescriptor!)!
```

修改 PipelineDescriptior 的 vertexDescriptor：

<pre class="language-swift"><code class="lang-swift">// MARK: - Descriptor
let pipelineDescriptor = MTL4RenderPipelineDescriptor()
pipelineDescriptor.vertexFunctionDescriptor        = vertexFunctionDescriptor
pipelineDescriptor.fragmentFunctionDescriptor      = fragmentFunctionDescriptor
<strong>pipelineDescriptor.vertexDescriptor                = .defaultLayout
</strong>pipelineDescriptor.colorAttachments[0].pixelFormat = .bgra8Unorm
</code></pre>

运行工程，你应该能看到一个灰色的盒子旋转了！

***

结束了么？让我们换一个 USDZ 模型试试：

{% file src="../../.gitbook/assets/Hammer (1).usdz" %}

{% code title="Renderer.swift" %}
```swift
let camera = Camera(
    position: SIMD3<Float>(0, 10, 15), // 调整了一下相机位置
    target: SIMD3<Float>(0, 0, 0),
    up: SIMD3<Float>(0, 1, 0)
)

let asset = AssetsLoader.loadAssets(named: "Hammer", ext: "usdz", device: device)!
```
{% endcode %}

当你把 asset 替换为 Hammer.usdz 时你会发现，整个模型完全加载不出来，这是因为这个模型拥有不止一个 Submesh，对于复杂的模型而言，需要在规范化且不会被回收的，保证在 Draw 的时候 GPU 还能访问到这些复杂的数据，这就要引出我们的 MTLResidencySet 显式 GPU 资源驻留机制

#### MTLResidencySet

MTLResidencySet 是苹果在 iOS 18+ 中引入的显式 GPU 资源管理模型，用于显式声明哪些资源（缓冲区、纹理、堆）在这个 Set（集合）内，对 GPU 可访问

以往只能依靠 `useResource` / `useHeap` 等 API，ResidencySet 是批量聚合的，只在 `commit()` 时生效

并且 ResidencySet 绑定到 **MTL4CommandBuffer** 或 **MTL4CommandQueue** 中，里面所有资源对该命令缓冲下的所有编码器都可用，不用反复声明

在 Renderer 中声明 ResidencySet：

<pre class="language-swift" data-title="Renderer.swift"><code class="lang-swift">// MARK: - Command Queue
let commandQueue: MTL4CommandQueue
let commandBuffer: MTL4CommandBuffer
let commandAllocator: MTL4CommandAllocator
<strong>let residencySet: MTLResidencySet
</strong>

init() {
    // MARK: - State
    // 创建渲染管线状态
    self.pipelineState = try device
        .makeCompiler(descriptor: MTL4CompilerDescriptor())
        .makeRenderPipelineState(descriptor: pipelineDescriptor)
    self.depthState = device
        .makeDepthStencilState(descriptor: depthStateDescriptor)!
<strong>    // 持久化资源
</strong><strong>    self.residencySet = try device
</strong><strong>        .makeResidencySet(descriptor: MTLResidencySetDescriptor())
</strong>    
    // MARK: - Command Queue
    self.commandQueue = device.makeMTL4CommandQueue()!
    self.commandBuffer = device.makeCommandBuffer()!
    self.commandAllocator = device.makeCommandAllocator()!
<strong>    // 绑定至命令队列
</strong><strong>    self.commandQueue.addResidencySet(residencySet)
</strong>}
</code></pre>

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

<pre class="language-swift" data-title="Renderer.swift"><code class="lang-swift">// MARK: - State
// 创建渲染管线状态
self.pipelineState = try device
    .makeCompiler(descriptor: MTL4CompilerDescriptor())
    .makeRenderPipelineState(descriptor: pipelineDescriptor)
self.depthState = device
    .makeDepthStencilState(descriptor: depthStateDescriptor)!
<strong>// 持久化资源
</strong><strong>self.residencySet = try device
</strong><strong>    .makeResidencySet(descriptor: MTLResidencySetDescriptor())
</strong><strong>self.residencySet.addAllocations(allocations)
</strong><strong>self.residencySet.commit()
</strong></code></pre>

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

点击导航栏中的 Hammer，会发现右侧检查器的 Transform 中，它们有三个属性：Position、Rotation、Scale，如果你拥有一些线性代数基础的话，你就会知道这里的三个属性，最终可以组合成一个 4×4 Transform Matrix，这就是它真正的 Model Matrix(模型矩阵)

展开发现有两个子节点叫 Circle 和 Cylinder，依次点击它们，从检查器中可以看到它们每个节点中，Transform 的值是不一样的，如果你尝试给 position、rotation 都设置为 0 的话，会得到：

<figure><img src="../../.gitbook/assets/Position 和 Rotation 都为 0 的 Hammer.png" alt="" width="375"><figcaption></figcaption></figure>

至此，你或许已经知道是怎么一回事了

因为 3D 模型使用节点化分层设计，假设我想让锤子旋转，我只需要对最外层的节点进行旋转就好了，分层设计还能明确职责，Mesh 只负责描述局部几何形状，节点负责描述整块 Mesh 在空间中的位置、旋转与缩放。

#### 再看看我们现在的代码是如何粗糙的处理 Model Matrix 的？

目前处理 Hammer 模型并没有去考虑它们父节点的 Transfom，并没有对齐进行正确的累加，所有的 Mesh 都在使用同一个 Model Matrix：

{% code title="Renderer.swift" %}
```swift
// 准备 MVP 矩阵
let modelMatrix = float4x4(rotationY: timer)

extension float4x4 {
    init(rotationY angle: Float) {
        self = float4x4(
            SIMD4<Float>( cos(angle), 0,  sin(angle), 0),
            SIMD4<Float>( 0,         1,  0,          0),
            SIMD4<Float>(-sin(angle), 0,  cos(angle), 0),
            SIMD4<Float>( 0,         0,  0,          1)
        )
    }
}
```
{% endcode %}

是时候使用它本身的 Transform 去构成正确的 Model Matrix 了！

#### 计算节点累加的 Transform

修改 Mesh 数据结构，让其支持存储 Transform：

{% code title="Mesh.swift" %}
```swift
import MetalKit

struct Mesh {
    let vertexBuffers: [MTLBuffer]
    let vertexBufferOffsets: [Int]
    let submeshes: [Submesh]
    let transform: float4x4
    
    init(mtkMesh: MTKMesh, transform: float4x4) {
        self.transform = transform
        vertexBuffers = mtkMesh.vertexBuffers.map { $0.buffer }
        vertexBufferOffsets = mtkMesh.vertexBuffers.map { $0.offset }
        
        submeshes = mtkMesh.submeshes.map { mtkSubmesh in
            Submesh(
                indexCount: mtkSubmesh.indexCount,
                indexType: mtkSubmesh.indexType,
                indexBuffer: mtkSubmesh.indexBuffer.buffer,
                indexBufferOffset: mtkSubmesh.indexBuffer.offset
            )
        }
    }
}
```
{% endcode %}

再明确一下接下来要做的事

1. 在遍历一个模型的所有 Mesh 时，需要把从根节点到当前节点路径上的 Transform 按顺序相乘，得到该节点在世界空间里的最终变换矩阵
2. 把这个矩阵和该 Mesh 一起收集起来，供渲染时把顶点从模型局部空间变换到世界空间使用。这里的“累加”就是矩阵连乘：父节点的世界矩阵乘上子节点的本地矩阵，得到子节点的世界矩阵。

那实际上对于 MDLObject 来说则是：深度遍历 MDLObject 的树结构，沿途维护 parentTransform，每到一个节点就取出 localTransform，再计算 worldTransform = parentTransform \* localTransform，哪怕 mesh 分散在不同层级、每层都有自己的平移/旋转/缩放

这样的行为也可以将其 **拍平 Flatten** 到 meshes 数组中，所以这里我们对 findAllMeshes 改名为 flattenMeshes：

<pre class="language-swift" data-title="Entity.swift"><code class="lang-swift">import MetalKit

class Entity {
    var name: String = "Untitled"
    var meshes: [Mesh]
    
    init(object: MDLObject, device: MTLDevice) {
        self.name = object.name
<strong>        let meshInfos = Self.flattenMeshes(object)
</strong>        
        let meshes: [Mesh] = meshInfos.compactMap { meshInfo in
            let mdlMesh = meshInfo.mesh
            let meshTransform = meshInfo.transform
            
            if mdlMesh.vertexDescriptor.attributeNamed(MDLVertexAttributeNormal) == nil {
                mdlMesh.addNormals(withAttributeNamed: MDLVertexAttributeNormal, creaseThreshold: 0.0)
            }
            
            guard let mtkMesh = try? MTKMesh(mesh: mdlMesh, device: device) else { return nil }
            
            return Mesh(mtkMesh: mtkMesh, transform: meshTransform)
        }
        
        self.meshes = meshes
    }
    
<strong>    static func flattenMeshes(
</strong><strong>        _ object: MDLObject,
</strong><strong>        parentTransform: float4x4 = matrix_identity_float4x4
</strong><strong>    ) -> [(mesh: MDLMesh, transform: float4x4)] {
</strong><strong>        var meshes: [(mesh: MDLMesh, transform: float4x4)] = []
</strong><strong>        let localTransform = object.transform?.matrix ?? matrix_identity_float4x4
</strong><strong>        let worldTransform = parentTransform * localTransform
</strong><strong>        
</strong><strong>        if let mesh = object as? MDLMesh {
</strong><strong>            meshes.append((mesh: mesh, transform: worldTransform))
</strong><strong>        }
</strong><strong>        
</strong><strong>        for child in object.children.objects {
</strong><strong>            meshes.append(contentsOf: flattenMeshes(child, parentTransform: worldTransform))
</strong><strong>        }
</strong><strong>        
</strong><strong>        return meshes
</strong><strong>    }
</strong>}
</code></pre>

此时，已经完成了对模型的正确加载

再回到 Renderer，需要重新思考 renderEntity 函数的逻辑，去读取每个 Mesh 的 Transform：

<pre class="language-swift" data-title="Renderer.swift"><code class="lang-swift">func renderEntity(_ entity: Entity, renderEncoder: MTL4RenderCommandEncoder) {
    for mesh in entity.meshes {
        guard !mesh.vertexBuffers.isEmpty else { continue }
<strong>        let modelMatrix = mesh.transform // 用这个 modelMatrix 去计算 MVP
</strong>        
    }
}
</code></pre>

但新问题出现了，目前写入 Uniforms 的逻辑是这样的：`memcpy(uniformBuffer.contents(), &uniforms, MemoryLayout.stride)` ，如果直接在循环中调用的话：

```swift
func renderEntity(_ entity: Entity, renderEncoder: MTL4RenderCommandEncoder) {
    for mesh in entity.meshes {
        let modelMatrix = sceneMatrix * mesh.transform
        updateUniforms(uniformBuffer, modelMatrix: modelMatrix)  // 每次都写入同一个位置
    }
}
```

对于这样的情况，我们要引入一个新概念：

#### Uniform Index

每个 Mesh 都需要一个与之对应的 Uniforms

还记得之前 [#nei-cun-bu-ju](../ru-men/ni-hao-san-jiao-xing.md#nei-cun-bu-ju "mention") 中介绍的 MTLBuffer 么？Buffer 本质上是一块连续的内存空间，只要对其合理的规划就能传递大量数据到 GPU 端：

| 第几个 Uniforms？ | 起始内存位置 | 内存大小 (字节) |
| ------------- | ------ | --------- |
| Uniforms\[0]  | 0      | 144       |
| Uniforms\[1]  | 144    | 144       |
| Uniforms\[2]  | 288    | 144       |

那只需要设定合适的 Offset \* 144 去作为读取 Buffer 的起始位置，就能让每个 Mesh 都能得到正确的 Uniforms，通过 MemoryLayout 计算得到 `MemoryLayout<Uniforms>.stride` 为 144，再定义一个 `uniformIndex`：

```swift
func renderEntity(
    _ entity: Entity,
    renderEncoder: MTL4RenderCommandEncoder,
    viewMatrix: float4x4,
    projectionMatrix: float4x4,
) {
    var uniformIndex = 0
    for mesh in entity.meshes {
        guard !mesh.vertexBuffers.isEmpty else { continue }
        let modelMatrix = mesh.transform
        // 计算这个 mesh 应该写到 buffer 的哪个位置
        let offset = uniformIndex * MemoryLayout<Uniforms>.stride
        // 下一个 mesh 要写到下一个位置
        uniformIndex += 1
        
        // 假定有这么一个 updateUniforms
        updateUniforms(uniformBuffer: uniformsBuffer, offset: offset, modelMatrix: modelMatrix)
    }
}
```

但再思考一下 🤔，如果有多个 Entity 的情况呢？看看外面是如何定义的：

```swift
for entity in entities {  // 可能有多个 Entity
    renderEntity(entity, renderEncoder: renderEncoder)
}
```

相当于每次都会创建一个 uniformIndex，会导致每个 Entity 内部的 `index` 都会从 0 重新开始，**不同 Entity 的 Mesh 数据会相互覆盖！**&#x6240;以需要将 uniformIndex 定义在外部

此处有两种解法：

{% tabs %}
{% tab title="每次绘制前重置" %}
<pre class="language-swift"><code class="lang-swift">class Renderer: NSObject, MTKViewDelegate {
    // MARK: - Buffers
    var uniformsBuffer: MTLBuffer
<strong>    var uniformIndex = 0 // 定义为属性
</strong>    
    func draw(in view: MTKView) {
        guard let drawable = view.currentDrawable else { return }
        timer += 0.005
<strong>        uniformIndex = 0 // 但是每帧绘制前将其重置
</strong>        
        for entity in entities {
            renderEntity(
                entity,
                renderEncoder: renderEncoder,
                viewMatrix: viewMatrix,
                projectionMatrix: projectionMatrix
            )
        }
    }
    
    func renderEntity(
        _ entity: Entity,
        renderEncoder: MTL4RenderCommandEncoder
    ) {
        for mesh in entity.meshes {
            guard !mesh.vertexBuffers.isEmpty else { continue }
            
            let modelMatrix = mesh.transform
            let offset = uniformIndex * MemoryLayout&#x3C;Uniforms>.stride
<strong>            uniformIndex += 1 // index +1
</strong>        }
    }
}
</code></pre>
{% endtab %}

{% tab title="局部变量" %}
<pre class="language-swift"><code class="lang-swift">func draw(in view: MTKView) {
    // MARK: - Draw
<strong>    var uniformIndex = 0 // 定义局部变量，仅这一次 draw 生效
</strong>    
    for entity in entities {
        renderEntity(
            entity,
            renderEncoder: renderEncoder,
<strong>            uniformIndex: &#x26;uniformIndex
</strong>        )
    }
    
    func renderEntity(
        _ entity: Entity,
        renderEncoder: MTL4RenderCommandEncoder,
<strong>        uniformIndex: inout Int // 使用 inout 关键字
</strong>    ) {
        for mesh in entity.meshes {
            guard !mesh.vertexBuffers.isEmpty else { continue }
            
            let modelMatrix = mesh.transform
            let offset = uniformIndex * MemoryLayout&#x3C;Uniforms>.stride
<strong>            uniformIndex += 1 // index +1
</strong>    }
}
</code></pre>
{% endtab %}
{% endtabs %}

选择任意一种即可

#### 为每个 Mesh 分配独立的 Uniform 数据 <a href="#headinge56c878afc1e44768595ba598276c040-wei-shen-me-xu-yao-wei-mei-ge-mesh-fen-pei-du-li-de-uniform" id="headinge56c878afc1e44768595ba598276c040-wei-shen-me-xu-yao-wei-mei-ge-mesh-fen-pei-du-li-de-uniform"></a>

在原来的 updateUniforms 函数里面还计算了 MVP，既然每个 Mesh 都有自己的 Model Matrix，那计算 MVP 矩阵就没法放在 `updateUniforms()` 函数中了，我们将其挪出来，同时让 updateUniforms 函数让其支持 offset：

<pre class="language-swift" data-title="Renderer.swift"><code class="lang-swift">func draw(in view: MTKView) {
    guard let drawable = view.currentDrawable else { return }
    timer += 0.005
<strong>    uniformIndex = 0
</strong>    
    // MARK: - 更新 Uniforms
    let aspect = view.drawableSize.width / view.drawableSize.height
<strong>    let viewMatrix = lookAt(
</strong><strong>        eye: camera.position,
</strong><strong>        target: camera.target,
</strong><strong>        up: camera.up
</strong><strong>    )
</strong><strong>    let projectionMatrix = perspective(
</strong><strong>        aspect: Float(aspect),
</strong><strong>        fovy: .pi / 3,
</strong><strong>        near: 0.1,
</strong><strong>        far: 1000
</strong><strong>    )
</strong>
    // MARK: - Draw
    for entity in entities {
        renderEntity(
            entity,
            renderEncoder: renderEncoder,
<strong>            viewMatrix: viewMatrix,
</strong><strong>            projectionMatrix: projectionMatrix
</strong>        )
    }
}

func renderEntity(
    _ entity: Entity,
    renderEncoder: MTL4RenderCommandEncoder,
<strong>    viewMatrix: float4x4,
</strong><strong>    projectionMatrix: float4x4
</strong>) {
    for mesh in entity.meshes {
        guard !mesh.vertexBuffers.isEmpty else { continue }
        
<strong>        let modelMatrix = mesh.transform
</strong><strong>        let offset = uniformIndex * MemoryLayout&#x3C;Uniforms>.stride
</strong><strong>        uniformIndex += 1
</strong>        
        updateUniforms(
            uniformsBuffer,
<strong>            offset: offset,
</strong><strong>            modelMatrix: modelMatrix,
</strong><strong>            viewMatrix: viewMatrix,
</strong><strong>            projectionMatrix: projectionMatrix
</strong>        )
    }
}

func updateUniforms(
    _ uniformBuffer: MTLBuffer,
<strong>    offset: Int,
</strong><strong>    modelMatrix: float4x4,
</strong><strong>    viewMatrix: float4x4,
</strong><strong>    projectionMatrix: float4x4
</strong>) {
    // 模型坐标 -> 世界坐标 -> 视图坐标 -> 裁剪坐标
    let mvpMatrix = projectionMatrix * viewMatrix * modelMatrix
    let normalMatrix = normalMatrix(modelMatrix: modelMatrix)
    
    var uniforms = Uniforms(
        mvpMatrix: mvpMatrix,
        normalMatrix: normalMatrix,
        light: light
    )
    
    // 复制到 GPU 缓冲区
<strong>    memcpy(uniformBuffer.contents().advanced(by: offset), &#x26;uniforms, MemoryLayout&#x3C;Uniforms>.stride)
</strong>}
</code></pre>

这里的 memcpy 函数中，可以通过添加 `advanced(by:)` 函数去设定起始位置，也就是 offset

#### 开始渲染

在前面，我们通过 `argumentTable.setAddress(mesh.vertexBuffers[0].gpuAddress, index: 0)` 简单实现了绘制单 Mesh 模型的行为，它的顶点/索引数据独占整个 Buffer。

但当模型包含多个 Mesh 时，Model I/O 会把它们的数据**合并到同一个 Buffer** 中。如果继续使用 `mesh.vertexBuffers[0].gpuAddress` 做读取的起始地址，所有 Mesh 都会从同一个位置开始读取数据，导致渲染出错。

但上面的修改 Mesh 数据结构中，已经用 `vertexBufferOffsets` 保存了每个 Buffer 的偏移量，对此只需要计算出 vertexBuffer 和 uniformsBufer 在此次循环中的地址就好了，修改：

<pre class="language-swift"><code class="lang-swift">updateUniforms(
    uniformsBuffer,
    offset: offset,
    modelMatrix: modelMatrix,
    viewMatrix: viewMatrix,
    projectionMatrix: projectionMatrix
)

<strong>// 使用之前记录的 mesh.vertexBufferOffsets[0]
</strong><strong>let vertexBufferAddress = mesh.vertexBuffers[0].gpuAddress + UInt64(mesh.vertexBufferOffsets[0])
</strong><strong>argumentTable.setAddress(vertexBufferAddress, index: 0)
</strong><strong>// 使用上方计算的此次循环中 UniformsBuffer 的 offset
</strong><strong>let uniformsBufferAddress = uniformsBuffer.gpuAddress + UInt64(offset)
</strong><strong>argumentTable.setAddress(uniformsBufferAddress, index: 1)
</strong>
for submesh in mesh.submeshes {
</code></pre>

至此，你应该能看到一个 Hammer 了！

但这次渲染只是碰巧我们使用的模型是每个 Submesh 都有自己独立 Buffer 或 offset 为 0，但这不代表这段绘制代码能渲染出所有的模型，对于多个 Submesh 共享一个 Buffer 的情况，这段代码就不太可用了

保险起见是通过 indexBuffer 起始位置加上每个 subMeshes 的偏移量计算出实际的起始位置，再通过总长度 `indexBuffer.length` 减去偏移量计算出这个 Buffer 的长度

<pre class="language-swift"><code class="lang-swift">func renderEntity(
    _ entity: Entity,
    renderEncoder: MTL4RenderCommandEncoder,
    viewMatrix: float4x4,
    projectionMatrix: float4x4
) {
    for mesh in entity.meshes {
        guard !mesh.vertexBuffers.isEmpty else { continue }
        let modelMatrix = mesh.transform
        let offset = uniformIndex * MemoryLayout&#x3C;Uniforms>.stride
        uniformIndex += 1
        
        updateUniforms(
            uniformsBuffer,
            offset: offset,
            modelMatrix: modelMatrix,
            viewMatrix: viewMatrix,
            projectionMatrix: projectionMatrix
        )
        
        // 使用之前记录的 mesh.vertexBufferOffsets[0]
        let vertexBufferAddress = mesh.vertexBuffers[0].gpuAddress + UInt64(mesh.vertexBufferOffsets[0])
        argumentTable.setAddress(vertexBufferAddress, index: 0)
        // 使用上方计算的此次循环中 UniformsBuffer 的 offset
        let uniformsBufferAddress = uniformsBuffer.gpuAddress + UInt64(offset)
        argumentTable.setAddress(uniformsBufferAddress, index: 1)
        
<strong>        for submesh in mesh.submeshes {
</strong><strong>            let indexBufferAddress = submesh.indexBuffer.gpuAddress + UInt64(submesh.indexBufferOffset)
</strong><strong>            let indexBufferLength = submesh.indexBuffer.length - submesh.indexBufferOffset
</strong><strong>            
</strong><strong>            renderEncoder.drawIndexedPrimitives(
</strong><strong>                primitiveType: .triangle,
</strong><strong>                indexCount: submesh.indexCount,
</strong><strong>                indexType: submesh.indexType,
</strong><strong>                indexBuffer: indexBufferAddress,
</strong><strong>                indexBufferLength: indexBufferLength
</strong><strong>            )
</strong><strong>        }
</strong>    }
}

</code></pre>

至此，无论是什么模型都能够正常渲染了。

<figure><img src="../../.gitbook/assets/一个 Hammer.png" alt="" width="375"><figcaption></figcaption></figure>

不过可以试个好玩的：在 model Matrix 上加一个 rotationY：

```swift
let modelMatrix = mesh.transform * float4x4(rotationY: timer)
```

会发现锤头和锤柄各自原地旋转，而不是作为一个整体旋转：

<figure><img src="../../.gitbook/assets/以自己为中心旋转的 Hammer.png" alt="" width="375"><figcaption></figcaption></figure>

这是因为 Model Matrix 是独自运用在一个局部位置，如果需要旋转，需要运用于整个 Entity



#### Transformable

对这些 Entity 来说，代表了一个 Asset 本体，通常会需要单独对它的 scale、position、rotation 进行单独设置，通过定义一个 Transformable 协议，方便后续快速读写它的 Transform 属性，来到 Model 下创建 Transform.swift:

{% code title="Transform.swift" expandable="true" %}
```swift
struct Transform {
    var position: SIMD3<Float> = .zero
    var rotation: SIMD3<Float> = .zero
    var scale: SIMD3<Float> = SIMD3<Float>(1, 1, 1)
    
    var modelMatrix: float4x4 {
        let translation = float4x4(translation: position)
        let rotation = float4x4(rotation: rotation)
        let scale = float4x4(scaling: scale)
        return translation * rotation * scale
    }
}

protocol Transformable {
    var transform: Transform { get set }
}

extension Transformable {
    var position: SIMD3<Float> {
        get { transform.position }
        set { transform.position = newValue }
    }
    
    var rotation: SIMD3<Float> {
        get { transform.rotation }
        set { transform.rotation = newValue }
    }
    
    var scale: SIMD3<Float> {
        get { transform.scale }
        set { transform.scale = newValue }
    }
}
```
{% endcode %}

为了方便后续编辑，创建 float4x4 的 extension 并删掉之前的 rotationY，创建 Extensions 路径并创建 float4x4+Extensions.swift

{% code title="float4x4+Extensions.swift" expandable="true" %}
```swift
extension float4x4 {
    init(translation t: SIMD3<Float>) {
        self = float4x4(
            SIMD4<Float>(1, 0, 0, 0),
            SIMD4<Float>(0, 1, 0, 0),
            SIMD4<Float>(0, 0, 1, 0),
            SIMD4<Float>(t.x, t.y, t.z, 1)
        )
    }
    
    init(scaling s: SIMD3<Float>) {
        self = float4x4(
            SIMD4<Float>(s.x, 0, 0, 0),
            SIMD4<Float>(0, s.y, 0, 0),
            SIMD4<Float>(0, 0, s.z, 0),
            SIMD4<Float>(0, 0, 0, 1)
        )
    }
    
    init(rotation r: SIMD3<Float>) {
        let rotationX = float4x4(
            SIMD4<Float>(1, 0, 0, 0),
            SIMD4<Float>(0, cos(r.x), sin(r.x), 0),
            SIMD4<Float>(0, -sin(r.x), cos(r.x), 0),
            SIMD4<Float>(0, 0, 0, 1)
        )
        
        let rotationY = float4x4(
            SIMD4<Float>(cos(r.y), 0, -sin(r.y), 0),
            SIMD4<Float>(0, 1, 0, 0),
            SIMD4<Float>(sin(r.y), 0, cos(r.y), 0),
            SIMD4<Float>(0, 0, 0, 1)
        )
        
        let rotationZ = float4x4(
            SIMD4<Float>(cos(r.z), sin(r.z), 0, 0),
            SIMD4<Float>(-sin(r.z), cos(r.z), 0, 0),
            SIMD4<Float>(0, 0, 1, 0),
            SIMD4<Float>(0, 0, 0, 1)
        )
        
        self = rotationZ * rotationY * rotationX
    }

    init(rotationX angle: Float) {
        self = float4x4(
            SIMD4<Float>(1, 0, 0, 0),
            SIMD4<Float>(0, cos(angle), sin(angle), 0),
            SIMD4<Float>(0, -sin(angle), cos(angle), 0),
            SIMD4<Float>(0, 0, 0, 1)
        )
    }
    
    init(rotationY angle: Float) {
        self = float4x4(
            SIMD4<Float>(cos(angle), 0, -sin(angle), 0),
            SIMD4<Float>(0, 1, 0, 0),
            SIMD4<Float>(sin(angle), 0, cos(angle), 0),
            SIMD4<Float>(0, 0, 0, 1)
        )
    }
    
    init(rotationZ angle: Float) {
        self = float4x4(
            SIMD4<Float>(cos(angle), sin(angle), 0, 0),
            SIMD4<Float>(-sin(angle), cos(angle), 0, 0),
            SIMD4<Float>(0, 0, 1, 0),
            SIMD4<Float>(0, 0, 0, 1)
        )
    }
}
```
{% endcode %}

让 Entity 遵循 Transformable 协议：

<pre class="language-swift" data-title="Entity.swift"><code class="lang-swift"><strong>class Entity: Transformable {
</strong>    var name: String = "Untitled"
    var meshes: [Mesh]
<strong>    var transform: Transform = Transform()
</strong>    
    ...
}
</code></pre>

最后使用上它，在 Renderer 的 renderEntity 函数中，修改 modelMatrix，将刚才加入的 transform 用上

```swift
let modelMatrix = entity.transform.modelMatrix * mesh.transform
```

随后，在 Draw 处，为 entity 添加旋转：

<pre class="language-swift"><code class="lang-swift">// MARK: - Draw
for entity in entities {
<strong>    entity.transform.rotation = SIMD3&#x3C;Float>(0, timer, 0)
</strong>    renderEntity(
        entity,
        renderEncoder: renderEncoder,
        viewMatrix: viewMatrix,
        projectionMatrix: projectionMatrix
    )
}
</code></pre>

至此，运行工程，应该就能看到多个 Mesh 的 Hammer 正在作为一个整体旋转了。
