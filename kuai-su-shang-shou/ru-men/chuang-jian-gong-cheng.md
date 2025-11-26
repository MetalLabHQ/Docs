# 创建工程

<table data-header-hidden><thead><tr><th width="100"></th><th width="100"></th></tr></thead><tbody><tr><td>作者</td><td>GH</td></tr></tbody></table>

Metal 4 是一个苹果生态的的图形 API，在开始学习 Metal 之前，确保你的设备是 Apple Silicon 的 Mac，并且系统更新到了 macOS Tahoe 26 及以上

本课程所有内容均考虑 iOS/macOS 等多平台，所以难免会写一些兼容性代码，同时在一些教程上，会在顶部留下折叠后的完整代码方便快速查阅。

#### 创建第一个工程：Hello World

**下载 Xcode**

前往 App Store，下载并安装 Xcode 最新版，并登录上自己的 Apple ID

**新建项目**

打开 Xcode，然后 **Create New Project** 创建一个新的项目，选择 **Multiplatform -> App**，之后只需填写项目名称（在这里我们填写MetalLab），其他选项默认即可，点击下一步后，可以自行决定是否勾选 **Create Git repository on my Mac**，如果勾选那么 Mac 会自动为工程创建 Git 仓库

#### 起步工程

此时有两个 .swift 文件：

* **HelloMetalViewApp.swift**：含 `@main` 的程序入口&#x20;
* **ContentView.swift**：视图部分，接下来将主要对该文件进行修改

在这之前，需要对 Apple 相关运行平台进行兼容，详情见 [duo-ping-tai-jian-rong-viewrepresentable.md](../../xiao-zhi-shi-dian/duo-ping-tai-jian-rong-viewrepresentable.md "mention")，创建 **PlatformCompatibility.swift** 文件：

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

这里的 **`typealias`** 是类似 C 语言中 **`typedef`** 的用法，给一个类型起一个别名来实现跨平台的应用开发。

运行当前项目，如果屏幕中央就会出现 Hello, world! 那么恭喜你创建了自己的第一个窗口！
