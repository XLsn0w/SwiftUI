# SwiftUI

SwiftUI 是一种非常简单的创新方法，可以利用 Swift 的强大能力在所有苹果设备平台上构建用户界面。通过 SwiftUI，开发者仅使用一组工具和 API 就能为所有苹果设备构建用户界面。SwiftUI 使用易于阅读和编写的声明式 Swift 语法，可与新的 Xcode 设计工具无缝协作，使你的代码和设计完美同步。SwiftUI 自动支持动态类型、黑暗模式、本地化和可访问性，你的 SwiftUI 代码将成为你写过的最强大的 UI 代码。


SwiftUI 最厉害的地方是其与 Xcode 深度集成，可以实时刷新预览，这将会改变 UI 的开发方式。另外其声明式语法写起来也挺方便。SwiftUI 的声明式语法，本身就是 Swift 的语法，属于语言内部 DSL。用了一些不太常见的语法特性，乍一看让人觉得很神奇。DSL(Domain Specific Language) 的概念见附录 1。

本文讨论 SwiftUI 所用到的不太常见语法特性。只讨论语法本身，SwiftUI 的意义，View 内部具体是如何渲染，之类的问题不会涉及。各小节内容如下，

```
some View

省略 return

链式调用

属性(Attribute)

@State，Property Delegates

尾随闭包(Trailing closure)

Function Builders

附录 1，DSL

附录 2，@dynamicMemberLookup 的实现流程

```

some View

```
struct ContentView : View {
    var body: some View {
        Text("Hello World")
    }
}
```


