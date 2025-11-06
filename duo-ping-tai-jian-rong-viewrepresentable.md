# 多平台兼容 ViewRepresentable

由于 macOS 使用 NextStep 继承下来的 AppKit，而 iOS/iPadOS/visionOS 等平台使用 UIKit，所以需要对 Apple 相关运行平台进行兼容，创建 **PlatformCompatibility.swift**：

{% code title="PlatformCompatibility.swift" %}
```swift
import SwiftUI

#if os(macOS)
typealias ViewRepresentable = NSViewRepresentable
typealias PlatformView = NSView
typealias PlatformViewController = NSViewController
typealias PlatformColor = NSColor
#else
typealias ViewRepresentable = UIViewRepresentable
typealias PlatformView = UIView
typealias PlatformViewController = UIViewController
typealias PlatformColor = UIColor
#endif
```
{% endcode %}
