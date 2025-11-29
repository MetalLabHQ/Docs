---
description: Hello, Renderer.
---

# 你好，渲染器

完整代码：

{% embed url="https://github.com/MetalLabHQ/HelloMetalView.git" %}

创建 Renderer.swift，我们要准备创建整个渲染器了。

Command Allocator 将 GPU 渲染指令编码成 Command Buffer，存入队列 Command Queue，GPU 会自动从 Command Queue 执行 Buffer 中的渲染指令：

{% embed url="https://www.figma.com/board/cqxWW0vQcBEFE4aeIUC7i9/%E5%91%BD%E4%BB%A4%E7%BC%93%E5%86%B2%E4%B8%8E%E9%98%9F%E5%88%97?node-id=0-1&t=5BUXEGTgwYVQWDkl-1" %}

**接下来要做什么？**

{% stepper %}
{% step %}
### 关联 Buffer 与 Allocator

1. Allocator 命令分配器获取可用 Command Buffer
2. 重置 Command Buffer 命令缓冲区状态
{% endstep %}

{% step %}
### 编码 Command Buffer

1. Command Buffer 初始化
2. 设定对应参数表
3. 创建渲染编码器 Render Command Encoder 并开始录制命令
4. 设置管线状态等参数
{% endstep %}

{% step %}
### 绘制图元，结束编码

1. 编码 GPU 绘制命令 Draw Primitives
2. Render Command Encoder 结束录制，Command Buffer 结束
3. 提交 Command Buffer 进入 Command Queue，GPU 会自动取出并执行 Buffer 内的绘制命令
4. GPU 渲染完成 Signal Drawable，呈现 `drawable.present()` 到 Metal View
5. 此时 Command Buffer 为可用状态，重复步骤 1，准备下一帧的内容
{% endstep %}
{% endstepper %}

#### 初始化渲染器 Renderer

将 Renderer 遵循 MTKViewDelegate，创建好命令队列、命令缓冲、命令分配器，

```swift
import SwiftUI
import MetalKit

class Renderer: NSObject, MTKViewDelegate {
    let device: MTLDevice                           // GPU 设备
    let commandQueue: MTL4CommandQueue              // 命令队列
    let commandBuffer: MTL4CommandBuffer            // Metal 命令 Buffer
    let commandAllocator: MTL4CommandAllocator      // 命令分配器

    init(device: MTLDevice) {
        self.device = device
        
        // MARK: - Command Queue
        self.commandQueue = device.makeMTL4CommandQueue()!
        self.commandBuffer = device.makeCommandBuffer()!
        self.commandAllocator = device.makeCommandAllocator()!
        
        super.init()
    }
    
    func draw(in view: MTKView) {
        
    }
    
    func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {}
}

#Preview {
    MetalView()
}
```

#### 开始绘制

Draw 是每一帧都会执行的函数，在这里要创建出一个让 GPU 渲染的指令

```swift
func draw(in view: MTKView) {
    guard let drawable = view.currentDrawable else { return }
    
    // MARK: - Begin Command Buffer
    self.commandQueue.waitForDrawable(drawable)
    self.commandAllocator.reset()
    self.commandBuffer.beginCommandBuffer(allocator: commandAllocator)
    
    
    // MARK: - End Command Buffer
    self.commandBuffer.endCommandBuffer()
    self.commandQueue.commit([commandBuffer], options: nil)
    self.commandQueue.signalDrawable(drawable)
    drawable.present()
}
```

1. 获取当前 drawable 作为画布并等待它回归可用状态
2. 重置命令分配器，Command Buffer 初始化
3. 执行渲染命令并提交到GPU
4. 呈现最终渲染结果到屏幕

这样整个命令缓冲区就提交到 GPU 了，接下来点击运行，你会得到一个——洋红色(Magenta)窗口

不用怕，洋红色是 GPU 出错时的标准调试颜色，这里只是因为 GPU 渲染了一个空的 Command Buffer，接下来为它添加一些渲染的内容就好了

在 `beginCommandBuffer` 和 `endCommandBuffer` 之间添加一个 **RenderPassDescriptor**，它是定义一次渲染操作的参数，这里做一个清屏操作：

```swift
let mtl4RenderPassDescriptor = MTL4RenderPassDescriptor()
// 渲染到屏幕纹理
mtl4RenderPassDescriptor.colorAttachments[0].texture = drawable.texture
// 渲染前清空画布
mtl4RenderPassDescriptor.colorAttachments[0].loadAction = .clear
// 清空所使用的颜色
mtl4RenderPassDescriptor.colorAttachments[0].clearColor  = MTLClearColor(red: 0.2, green: 0.2, blue: 0.25, alpha: 1.0)
```

继续准备 **Render Command Encoder**，它用于录制渲染命令，但我们啥也不录制

```swift
guard let renderEncoder = commandBuffer.makeRenderCommandEncoder(descriptor: mtl4RenderPassDescriptor, options: MTL4RenderEncoderOptions()) else { return }
renderEncoder.endEncoding()
```

至此，点击运行应该能看见一个清屏颜色的背景了。

延伸阅读：[ming-ling-dui-lie-command-queue](../../ming-ling-dui-lie-command-queue/ "mention")
