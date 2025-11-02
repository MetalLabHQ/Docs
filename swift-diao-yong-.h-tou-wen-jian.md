# Swift 调用 .h 头文件

前往 Xcode Build Settings

<figure><img src=".gitbook/assets/Xcode 中的 Build Settings 位置.png" alt=""><figcaption><p>Build Settings 位置</p></figcaption></figure>

在右上角搜索框内搜索：**Objective-C Bridging Header**

双击当前 Target 的一栏，例如 Target 叫 HelloTriangle 就在对应栏位的位置双击并输入：

`$(SRCROOT)/$(TARGET_NAME)/.h头文件路径`

或者也可以自己定制路径，这里是一些特殊标记，用于辅助输入相对路径：

$(SRCROOT)：工程路径

$(TARGET\_NAME)：Target 名称

$(PRODUCT\_NAME)：项目（产品）名称
