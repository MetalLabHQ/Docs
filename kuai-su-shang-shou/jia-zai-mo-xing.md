# 加载模型

完整代码：



本章节会教你如何真正的加载一个模型，

下载该工程作为起点：

{% file src="../.gitbook/assets/LoadAssets.7z" %}
初始工程
{% endfile %}

#### 发生了什么？

删掉了 Index 与 Vertex 数据结构，因为我们不再需要手动编写模型顶点数据，也删掉了与之对应的 Buffer，目前不需要

这一次我们要根据模型的实际情况来决定内存布局了，所以删掉了 Vertex Descriptor

#### 准备 3D 模型

我们需要一个能渲染的模型，截角二十面体，下载该文件并放入 Assets 文件夹：

{% file src="../.gitbook/assets/truncated icosahedron.obj" %}

<details>

<summary>截角二十面体</summary>

在几何学中，截角二十面体是一种由12个正五边形和20个正六边形所组成的凸半正多面体，也是足球的模样，同时具有每个三面角等角和每条边等长的性质，因此属于阿基米德立体，但由于其并非所有面全等因此不能算是正多面体。由于其包含了正五边形和六边形面，因此也是一种戈德堡多面体，其对偶多面体为五角化十二面体。

</details>

这是一个 obj 文件，这是最基本的 3D 文件格式，省去了我们手动定义顶点的时间，并且它也能携带一些少量信息如法线、纹理坐标等，详情见 [.obj.md](../3d-wen-jian-ge-shi-su-cha/.obj.md "mention")

#### 加载模型

在偏现代化一些的格式如 fbx、gltf、usd 中，但这里我们暂且先不使用一个文件含有多个 Mesh 的文件，聚焦于这个单一 Mesh 的 obj 文件，不过我们依旧给 Mesh 设计成数组，在 Models 下创建 Entity.swift

{% code title="Entity.swift" %}
```swift
import MetalKit

class Entity {
    var name: String = "Untitled"
    var meshes: [Mesh]
    
    init(object: MDLObject, device: MTLDevice) {
        self.name = object.name
        let mdlMeshes = Self.findAllMeshes(object)
        let mtkMeshes = mdlMeshes.compactMap { try? MTKMesh(mesh: $0, device: device) }
        self.meshes = mtkMeshes.map(Mesh.init)
    }
    
    static func findAllMeshes(_ object: MDLObject) -> [MDLMesh] {
        var meshes: [MDLMesh] = []
        if let mesh = object as? MDLMesh {
            meshes.append(mesh)
        }
        for child in object.children.objects {
            meshes.append(contentsOf: findAllMeshes(child))
        }
        return meshes
    }
}
```
{% endcode %}

从前面的学习，我们了解到模型是由一个个顶点组成多个三角面来绘制在屏幕上的，而对于模型而言，这样一个连接起来的模型称为一个 Mesh（网格）

{% code title="Entity.swift" %}
```swift
struct Mesh {
    var vertexBuffers: [MTLBuffer]
    var submeshes: [Submesh]
    
    init(mtkMesh: MTKMesh) {
        vertexBuffers = mtkMesh.vertexBuffers.map { $0.buffer }
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

struct Submesh {
    let indexCount: Int
    let indexType: MTLIndexType
    let indexBuffer: MTLBuffer
    let indexBufferOffset: Int
}
```
{% endcode %}

使用苹果开发等 Model I/O Framework，可以轻松读取行业通用的模型格式，转换为 MDLAssets，节省了手动解析模型的时间，创建 Module 文件夹并创建 AssetsLoader.swift 文件，有兴趣可以自己手动使用 printAssetInfo 来查看模型内部结构，也可以放一些你自己的模型格式进去试试

{% tabs %}
{% tab title="loadAssets" %}
{% code title="AssetsLoader.swift" %}
```swift
enum AssetsLoader {
    static func loadAssets(named filename: String, ext: String, device: MTLDevice) -> MDLAsset? {
        guard let url = Bundle.main.url(forResource: filename, withExtension: ext) else {
            print("Cannot find file: \(filename).\(ext)")
            return nil
        }
        
        if !MDLAsset.canImportFileExtension(ext) {
            print("File extension .\(ext) is not supported for import")
        }
        
        let asset = MDLAsset(
            url: url,
            vertexDescriptor: nil,
            bufferAllocator: MTKMeshBufferAllocator(device: device)
        )
        
        return asset
    }
}
```
{% endcode %}
{% endtab %}

{% tab title="printAssetInfo" %}
{% code title="AssetsLoader.swift" expandable="true" %}
```swift
import MetalKit

enum AssetsLoader {
    static func loadAssets(named filename: String, ext: String, device: MTLDevice) -> MDLAsset? {
        guard let url = Bundle.main.url(forResource: filename, withExtension: ext) else {
            print("Cannot find file: \(filename).\(ext)")
            return nil
        }
        
        if !MDLAsset.canImportFileExtension(ext) {
            print("File extension .\(ext) is not supported for import")
        }
        
        let asset = MDLAsset(
            url: url,
            vertexDescriptor: nil,
            bufferAllocator: MTKMeshBufferAllocator(device: device)
        )
        
        return asset
    }
    
    static func printAssetInfo(asset: MDLAsset) {
        print("\n========== Asset Info ==========")
        let bbox = asset.boundingBox
        print("  bounding_box:")
        let formatVec = { (v: SIMD3<Float>) in String(format: "%.6f, %.6f, %.6f", v.x, v.y, v.z) }
        print("    min: [\(formatVec(bbox.minBounds))]")
        print("    max: [\(formatVec(bbox.maxBounds))]")
        print("    extent: [\(formatVec(bbox.maxBounds - bbox.minBounds))]")
        print("\nobjects:")
        
        for index in 0..<asset.count {
            printObjectInfo(object: asset.object(at: index), depth: 0, isFirstInList: index == 0)
        }
    }
    
    private static func printObjectInfo(object: MDLObject, depth: Int, isFirstInList: Bool = false) {
        let i = String(repeating: "    ", count: depth)
        let type = String(describing: type(of: object))
        
        if !isFirstInList { print() }
        
        print("\(i)> name: \"\(object.name)\"")
        print("\(i)  type: \(type)")
        
        if let transform = object.transform {
            let matrix = transform.matrix
            let pos = SIMD3<Float>(matrix.columns.3.x, matrix.columns.3.y, matrix.columns.3.z)
            let scale = SIMD3<Float>(
                length(SIMD3<Float>(matrix.columns.0.x, matrix.columns.0.y, matrix.columns.0.z)),
                length(SIMD3<Float>(matrix.columns.1.x, matrix.columns.1.y, matrix.columns.1.z)),
                length(SIMD3<Float>(matrix.columns.2.x, matrix.columns.2.y, matrix.columns.2.z))
            )
            
            let eps: Float = 0.0001
            let isDefault = abs(pos.x) < eps && abs(pos.y) < eps && abs(pos.z) < eps &&
            abs(scale.x - 1.0) < eps && abs(scale.y - 1.0) < eps && abs(scale.z - 1.0) < eps
            
            if !isDefault {
                print("\(i)  transform:")
                let formatVec = { (v: SIMD3<Float>) in String(format: "%.6f, %.6f, %.6f", v.x, v.y, v.z) }
                print("\(i)    position: [\(formatVec(pos))]")
                print("\(i)    scale: [\(formatVec(scale))]")
            }
        }
        
        if let mesh = object as? MDLMesh {
            print("\(i)  mesh:")
            print("\(i)    vertex_count: \(mesh.vertexCount)")
            print("\(i)    submesh_count: \(mesh.submeshes?.count ?? 0)")
            
            let bbox = mesh.boundingBox
            print("\(i)    bounding_box:")
            let formatVec = { (v: SIMD3<Float>) in String(format: "%.6f, %.6f, %.6f", v.x, v.y, v.z) }
            print("\(i)      min: [\(formatVec(bbox.minBounds))]")
            print("\(i)      max: [\(formatVec(bbox.maxBounds))]")
            
            let attrs = mesh.vertexDescriptor.attributes.compactMap { $0 as? MDLVertexAttribute }
                .filter { $0.format != .invalid }
            
            if !attrs.isEmpty {
                print("\(i)    vertex_attributes:")
                let attrsByBuffer = Dictionary(grouping: attrs) { $0.bufferIndex }
                for bufferIndex in attrsByBuffer.keys.sorted() {
                    guard let bufferAttrs = attrsByBuffer[bufferIndex] else { continue }
                    var stride = 0
                    if bufferIndex < mesh.vertexDescriptor.layouts.count,
                       let layout = mesh.vertexDescriptor.layouts[bufferIndex] as? MDLVertexBufferLayout {
                        stride = layout.stride
                    }
                    print("\(i)      buffer[\(bufferIndex)] (stride: \(stride) bytes):")
                    for attr in bufferAttrs.sorted(by: { $0.offset < $1.offset }) {
                        print("\(i)        - name: \"\(attr.name)\"")
                        print("\(i)          format: \"\(attr.format.displayName)\"")
                        print("\(i)          offset: \(attr.offset)")
                    }
                }
            }
        }
        
        let children = object.children.objects
        if !children.isEmpty {
            print("\(i)  children: [")
            for (index, child) in children.enumerated() {
                printObjectInfo(object: child, depth: depth + 1, isFirstInList: index == 0)
            }
            print("\(i)  ]")
        }
    }
}

extension MDLVertexFormat {
    var displayName: String {
        switch self {
        case .invalid: return "Invalid"
        case .float4: return "Float4"
        case .float3: return "Float3"
        case .float2: return "Float2"
        case .float: return "Float"
        case .int4: return "Int4"
        case .int3: return "Int3"
        case .int2: return "Int2"
        case .int: return "Int"
        case .uChar4: return "UnsignedChar4"
        case .uChar3: return "UnsignedChar3"
        case .uChar2: return "UnsignedChar2"
        case .uChar: return "UnsignedChar"
        case .uChar4Normalized: return "UnsignedChar4Normalized"
        case .uChar3Normalized: return "UnsignedChar3Normalized"
        case .uChar2Normalized: return "UnsignedChar2Normalized"
        case .uCharNormalized: return "UnsignedCharNormalized"
        case .char4: return "Char4"
        case .char3: return "Char3"
        case .char2: return "Char2"
        case .char: return "Char"
        case .char4Normalized: return "Char4Normalized"
        case .char3Normalized: return "Char3Normalized"
        case .char2Normalized: return "Char2Normalized"
        case .charNormalized: return "CharNormalized"
        case .uShort4: return "UnsignedShort4"
        case .uShort3: return "UnsignedShort3"
        case .uShort2: return "UnsignedShort2"
        case .uShort: return "UnsignedShort"
        case .uShort4Normalized: return "UnsignedShort4Normalized"
        case .uShort3Normalized: return "UnsignedShort3Normalized"
        case .uShort2Normalized: return "UnsignedShort2Normalized"
        case .uShortNormalized: return "UnsignedShortNormalized"
        case .short4: return "Short4"
        case .short3: return "Short3"
        case .short2: return "Short2"
        case .short: return "Short"
        case .short4Normalized: return "Short4Normalized"
        case .short3Normalized: return "Short3Normalized"
        case .short2Normalized: return "Short2Normalized"
        case .shortNormalized: return "ShortNormalized"
        case .half4: return "Half4"
        case .half3: return "Half3"
        case .half2: return "Half2"
        case .half: return "Half"
        default: return "Format(\(self.rawValue))"
        }
    }
}
```
{% endcode %}
{% endtab %}
{% endtabs %}



