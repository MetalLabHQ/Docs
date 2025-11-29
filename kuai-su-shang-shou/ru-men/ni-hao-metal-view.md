---
description: Hello, Metal View
---

# 你好，Metal View

在开始渲染前，我们需要准备一个专门显示 Metal 画面的视图，也就是 Metal View

Metal View 封装了 Metal 渲染所需的基础设置，让你能够快速开始 Metal 开发而无需处理复杂的底层配置，类似于 OpenGL 中的 GLFW 窗口。

修改项目中的 ContentView，右键点击编辑区代码中的 ContentView，选择 Refactor -> Rename，将其改名为 MetalView，并将代码改为：

<details>

<summary>MetalView 完整代码</summary>

{% code title="MetalView.swift" %}
```swift
import SwiftUI
import MetalKit

struct MetalView: ViewRepresentable {
    let device: MTLDevice
    let renderer: Renderer
    
    init() {
        guard let defaultDevice = MTLCreateSystemDefaultDevice() else {
            fatalError("Metal is not supported")
        }
        
        guard defaultDevice.supportsFamily(.metal4) else {  // 检查是否支持 Metal 4
            fatalError("Metal 4 is not supported on this device")
        }
        
        device = defaultDevice
        do {
            self.renderer = try Renderer(device: device)
        } catch {
            fatalError("Error: \(error)")
        }
    }
    
#if os(macOS)
    func makeNSView(context: Context) -> MTKView {
        return makeView()
    }
    
    func updateNSView(_ nsView: MTKView, context: Context) {}
#else
    func makeUIView(context: Context) -> MTKView {
        return makeView()
    }
    
    func updateUIView(_ uiView: MTKView, context: Context) {}
#endif
    
    func makeView() -> MTKView {
        let mtkView = MTKView(frame: .zero, device: device)
        // 设置渲染器代理，分离 UI 和 Renderer
        mtkView.delegate = renderer
        // 像素格式
        mtkView.colorPixelFormat = .bgra8Unorm
        // 清屏颜色
        mtkView.clearColor = MTLClearColor(red: 0.0, green: 0.0, blue: 0.0, alpha: 1.0)
        return mtkView
    }
}

#Preview {
    MetalView()
}
```
{% endcode %}

</details>

#### 详细解读

**属性定义**

```swift
let device: MTLDevice
let renderer: Renderer
```

* MTLDevice 是 Metal 的核心对象，代表 GPU 设备，所有 Metal 操作都需要通过它进行
* Renderer 则是我们接下来会手动定义的渲染器本体，这里先不用管

**初始化 Metal View**

```swift
init() {
    guard let defaultDevice = MTLCreateSystemDefaultDevice() else {
        fatalError("Metal is not supported")
    }

    guard defaultDevice.supportsFamily(.metal4) else {  // 检查是否支持 Metal 4
        fatalError("Metal 4 is not supported on this device")
    }

    device = defaultDevice
    do {
        self.renderer = try Renderer(device: device)
    } catch {
        fatalError("Error: \(error)")
    }
}
```

* 获取并创建系统默认的 **MTLDevice**
* **检查是否支持 Metal 4，本教程为 Metal 4 的教程，一定程度上与 Metal 3 存在差异**
* Renderer 则是我们接下来会手动定义的渲染器本体，负责具体的渲染逻辑，这里先不用管

**遵循 ViewRepresentable 协议，处理平台兼容性**

```swift
struct MetalView: ViewRepresentable {
#if os(macOS)
    func makeNSView(context: Context) -> MTKView {
        return makeView()
    }
    
    func updateNSView(_ nsView: MTKView, context: Context) {}
#else
    func makeUIView(context: Context) -> MTKView {
        return makeView()
    }
    
    func updateUIView(_ uiView: MTKView, context: Context) {}
#endif
}
```

* macOS 使用 `NSViewRepresentable`，需要实现 `makeNSView` 和 `updateNSView`&#x20;
* iOS/iPadOS 等使用 `UIViewRepresentable`，需要实现 `makeUIView` 和 `updateUIView`
* 通过条件编译 `#if os(macOS)` 来区分不同平台

**创建 Metal View 视图**

```swift
func makeView() -> MTKView {
    let mtkView = MTKView(frame: .zero, device: device)
    // 设置渲染器代理，分离 UI 和 Renderer
    mtkView.delegate = renderer
    // 像素格式
    mtkView.colorPixelFormat = .bgra8Unorm
    // 清屏颜色
    mtkView.clearColor = MTLClearColor(red: 0.0, green: 0.0, blue: 0.0, alpha: 1.0)
    return mtkView
}
```

* MTKView 是 Apple 提供的 Metal 视图容器
* colorPixelFormat 设置颜色缓冲区格式，bgra8Unorm 是大多数设备都支持的标准格式
* mtkView.delegate = renderer 将渲染器设置为代理，MTKView 会在需要渲染时自动调用渲染器的方法
* clearColor 则是每次渲染前清屏的颜色，这里设置为纯黑色

至此，点击运行，应该可以看见整个空着的 MetalView 了
