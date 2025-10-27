# MTKView

MTKView 是渲染目标（画布），负责显示渲染结果，推荐 [#jiang-mtkview-feng-zhuang-cheng-metalview](mtkview.md#jiang-mtkview-feng-zhuang-cheng-metalview "mention")

&#x20;makeView 中使用 frame: .zero 使视图尺寸由 SwiftUI 管理，并指定 MTLDevice

<details>

<summary>将 MTKView 封装成 MetalView</summary>

使用 [kua-ping-tai-jian-rong.md](kua-ping-tai-jian-rong.md "mention") 封装的 ViewRepresentable 协议

分离渲染部分 Renderer 作为 Delegate ，而 MetalView 处理 UI 相关部分

```swift
import SwiftUI
import MetalKit

struct MetalView: ViewRepresentable {
    // MARK: - Metal Device
    private let device: MTLDevice
    private let renderer: Renderer
    
    init() {
        guard let defaultDevice = MTLCreateSystemDefaultDevice() else {
            fatalError("Metal is not supported")
        }
        
        // 检查 Metal 4 支持
        guard defaultDevice.supportsFamily(.metal4) else {
            fatalError("Metal 4 is not supported on this device")
        }
        self.device = defaultDevice
        
        do {
            self.renderer = try Renderer(device: device)
        } catch {
            print("Failed to create Metal 4 renderer: \(error)")
            fatalError("Failed to create Metal 4 renderer: \(error)")
        }
    }
    
    // MARK: - 创建视图
#if os(macOS)
    func makeNSView(context: Context) -> MTKView {
        return makeView()
    }
    
    func updateNSView(_ nsView: MTKView, context: Context) {
        // 更新逻辑
    }
#else
    func makeUIView(context: Context) -> MTKView {
        return makeView()
    }
    
    func updateUIView(_ uiView: MTKView, context: Context) {
        // 更新逻辑
    }
#endif
    
    // 共享的视图创建逻辑
    private func makeView() -> MTKView {
        let mtkView = MTKView(frame: .zero, device: device)
        // 配置渲染格式
        mtkView.colorPixelFormat = .bgra8Unorm
        // 设置渲染器代理，分离 UI 和 Renderer
        mtkView.delegate = renderer
        // 配置其他渲染属性
        mtkView.clearColor = MTLClearColor(red: 0.0, green: 0.0, blue: 0.0, alpha: 1.0)
        return mtkView
    }
}
```

</details>

#### 创建 MTKView

makeView 中使用 frame: .zero 使视图尺寸由 SwiftUI 管理，并指定 MTLDevice

```swift
MTKView(frame: .zero, device: device)
```

设置颜色格式为标准化的 BGRA 8 位格式，这也是常用的渲染像素格式

```swift
mtkView.colorPixelFormat = .bgra8Unorm
```

<details>

<summary>Q: <strong>传入 MTLDevice 好还是初始化时生成好？</strong></summary>

初始化阶段时生成适合整个应用只有一出需要用到 MetalView 的情况，如果多处使用请管理好 MTLDevice 实例，使用单例模式或外部传入 MTLDevice 会更保险

```swift
struct MetalView: UIViewRepresentable {
    private let device: MTLDevice
    private let renderer: Renderer
    
    init() {
        self.device = MTLCreateSystemDefaultDevice()!
        self.renderer = Renderer(device: device)
    }
}
```

</details>

