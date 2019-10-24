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


SwiftUI 的 View 是对于 UI 应该是如何展示的一个数据描述，并非真正用于显示的 View。现在的 iOS，底层会用 UIKit 实现，最终从数据描述的 View 生成真正的 UIView。

每个 View 的内容，就是其 body 属性。返回值为 some View，这里的 some 需要解释一下。

```
public protocol View : _View {
    associatedtype Body : View
    var body: Self.Body { get }
}
```

SwiftUI 的 View 实现成协议，使用 associatedtype 关联了一个 Body 类型。根据 Swift 的语法，带有 associatedtype 的协议不能直接返回，只能作为类型约束。

// Error
// Protocol 'View' can only be used as a generic constraint
// because it has Self or associated type requirements
func createView() -> View {
}

// OK
func createView<T: View>() -> T {
}

在 Swift 5.1 之前，要绕开 associatedtype 的限制，需要明确写出真实类型。body 需要写成。
```
struct ContentView : View {
    var body: Text {
        Text("Hello World")
    }
}
```
但这样写的话，每次修改了 body 的返回值（比如将 Text 修改成 Button)，就需要手动修改相应的类型，会很麻烦。其实我们真实关心返回值是否是 View, 不关心到底是 Text 还是 Button，不能这样写只是语法的限制。

为绕开这个限制，Swift 5.1 引入了 Opaque Result Types[2]。写成 some View 就保证返回值是个确定的 View，让编译器网开一面。值得注意的是，返回类型必须是确定的，比如下面分支代码，不同分支返回不同的类型，就会编译出错。

```
// Error
// Function declares an opaque return type, but the return 
// statements in its body do not have matching underlying types
let someCondition: Bool = false

var body: some View {
    if someCondition {
        return Text("Hello World")
    } else {
        return Button(action: {}) {
            Text("Tap me")
        }
    }
}
```

省略 return
```
struct ContentView : View {
    var body: some View {
        Text("Hello World")
    }
}
```

上面 body 的写法还有个小细节，body 的返回值为 View，但是代码当中并没有写 return, 似乎并没有返回值。

这是 Swift 的语法特性，见 SE-0255[3]。当函数体中只有单独一个表达式，就会自动添加一个 return，返回这个表达式的值。上述代码，相当于

struct ContentView : View {
    var body: some View {
        return Text("Hello World")
    }
}

注意，只有函数体中是单独表达式，才会自动添加 return。下面的代码有两个表达式，是错的。

// Error
// Function declares an opaque return type, but has 
// no return statements in its body from which to infer an underlying type

```
struct ContentView : View {
    var body: some View {
        Text("Hello World")
        Text("Hello World")
    }
}
```
另外单独的表达式并不代表就只有一行代码。例子中 ZStack、VStack、HStack 常写成多行，但只是一个表达式。

假如用语法树来说，就是函数体的语法树只有一个子节点元素。见 ParseDecl.cpp

// If the body consists of a single expression, turn it into a return
// statement.
//
// But don't do this transformation during code completion, as the source
// may be incomplete and the type mismatch in return statement will just
// confuse the type checker.
if (!Body.hasCodeCompletion() && BS->getNumElements() == 1) {

链式调用
```
struct ContentView : View {
    var body: some View {
        Text("Hello World")
            .bold()
            .font(.title)
    }
}
```

上述写法连续用 bold, font 等函数来修改 Text 的值，这种写法叫链式调用。实现的方式类似下面，通常是修改了数据后，返回自身。

```
struct MyText {
    init(_ str: String) {
        print("init")
    }

    func bold() -> MyText {
        print("change bold")
        return self
    }

    func font(_ font: Font?) -> MyText {
        print("change font")
        return self
    }
}
```

// 可以写成
MyText("Hello World").bold().font(.title)

OC 也可以实现这种点语法的连续调用，但会麻烦很多。其实不一定返回自身，只要返回一个对象就可以使用点语法了，只是返回自身很常见。比如下面代码返回不同对象：
```
extension Int {
    var hours : Date {
        return Date(timeIntervalSinceNow: TimeInterval(self) * 3600.0)
    }

    var days : Date {
        return Date(timeIntervalSinceNow: TimeInterval(self) * 3600.0 * 24.0)
    }
}

extension Date {
    var ago : Date {
        return Date(timeIntervalSinceNow: -self.timeIntervalSinceNow);
    }
}
```

// 可以写成
3.days.ago
3.hours.ago

属性(Attribute)
接下来，我们应该讲述下面代码中 @State 那个语法。
```
struct RoomDetail : View {
    @State var zoomed = false
}

但讲述前，先岔开一下，说一下 Attribute。在中文中，Property 和 Attribute 都被翻译成属性了。但两者在 Swift 中是不同的概念。

struct FixedLengthRange {
    var firstValue: Int  // Property
    let length: Int
}

@available(iOS 10.0, macOS 10.12, *) // Attribute
class MyClass {
    // class definition
}
```

在 Swift 中，Property 是指对象中，firstValue、length 这种语法。而 Attribute 是指 @ 字符开头的，类似 @available 这种语法。为了不产生误导，下文会使用英文术语。

Swift 中有各种 Attribute，比如

@dynamicCallable
@dynamicMemberLookup
@available
@objc

Swift 的 Attribute 语法可以放到类型定义或者函数定义的前面，是对类型和函数的一种标记。

下面大致描述 Attribute 的原理，具体的实现细节可能会有出入。

编译 Swift 源代码时，在解析阶段（Prase）, 会生成一个抽象语法树（AST，Abstract Syntax Tree)。语法树生成时，所有的 Attribute 统一处理，生成 Attribute 节点。之后在语义分析阶段(semantic analysis)，会有可能触发 Attribute 节点，使其对语法树本身产生影响。

不同的 Attribute 对语法树可以有不同的影响。比如 @available 会根据系统对语法树的函数调用进行可行性检查，不修改语法树本身。而 @dynamicMemberLookup，@dynamicCallable 进行检查后，可能会直接修改语法树本身，从而转调某些根据规则命名好的类或者函数。

Attribute 是种元编程（Metaprogramming）手段，Attribute 语法会被编译成语法树节点，而 Attribute 又可以反过来修改语法树本身。在类定义或函数定义前添加不同的 Attribute，可以不同的方式修改语法树，从而实现某些常规方式难以实现的语法。其实很好理解，既然都可以修改语法树了，自然就可以通过 Attribute 实现神奇的语法。

假如修改 Swift 的源码，可以根据不同的场合，很容易添加自定义 Attribute。比如 @UIApplicationMain 就是一个自定义 Attribute 扩展，为语法树添加了入口 main 函数。因而用 swift 写 iOS App, 是不用自己写 main 函数的。

本文附录 2，有 @dynamicMemberLookup 的实现流程，跟 Swift 语法本身没有太大关系，但可增加对 Attribute 语法的理解。

@State，Property Delegates
struct RoomDetail : View {
    @State var zoomed = false
}

现在可以来讨论 @State 这个语法了，SwiftUI 用 @State 来维护状态，状态改变后，会自动更新 UI。类似的语法还有 @Binding，@@Environment 等。

这个语法特性看起来很神奇，叫 Property Delegates[4]。

State 其实只是个自定义类，用 @propertyDelegate 修饰，将 zoomed 的读写转到 State 实现了。其余的 @Binding，@Environment 一样的道理，将 Property 读写转到 Binding 和 Environment 类实现了。

@propertyDelegate public struct State<Value>
@propertyDelegate public struct Binding<Value>
@propertyDelegate public struct Environment<Value>

我们先来弄明白为什么需要 Property Delegates。

举个例子，假设我有个 App, 需要在程序首次启动时，显示一个帮助信息。我将是否首次启动记录在 UserDefaults 中。为了防止写错 Key, 我可以这样实现。

struct GlobalSettings {
    static var isFirstLanch: Bool {
        get {
            return UserDefaults.standard.object(forKey: "isFirstLanch") as? Bool ?? false
        } set {
            UserDefaults.standard.set(newValue, forKey: "isFirstBoot")
        }
    }
}

GlobalSettings.isFirstLanch 调用了 UserDefaults.standard 的实现。这时我又想在 UserDefaults 保存另一个信息会怎么办呢，比如字体大小。复制粘贴修改，我们可以写成

struct GlobalSettings {
    static var uiFontValue: Float {
        get {
            return UserDefaults.standard.object(forKey: "uiFontValue") as? Float ?? 14
        } set {
            UserDefaults.standard.set(newValue, forKey: "uiFontValue")
        }
    }
}

可以看到 GlobalSettings.isFirstLanch 跟 GlobalSettings.uiFontValue 两者代码重复了。假如要保存多个值，就会重复 多次。为了避免重复代码，可以将相同的行为指派某个代理对象去做，为此引入 Property Delegates。

@propertyDelegate
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    var value: T {
        get {
            return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}


struct GlobalSettings {
    @UserDefault(key: "isFirstLanch", defaultValue: false)
    static var isFirstLanch: Bool

    @UserDefault(key: "uiFontValue", defaultValue: 14)
    static var uiFontValue: Float
}

使用 @propertyDelegate 修饰了 UserDefault, 它就可以作为代理对象。之后使用 @UserDefault 去修饰 Property，会自动定义出这个代理对象，将实现转到这个对象了。将 isFirstLanch 展开，相当于：

struct GlobalSettings {
    static var $isFirstLanch = UserDefault<Bool>(key: "isFirstLanch", defaultValue: false)
    static var isFirstLanch: Bool {
        get {
            return $isFirstLanch.value
        }
        set {
            $isFirstLanch.value = newValue
        }
    }
}

同理 @State 这个语法，也是 Property Delegates，将下面代码展开，会变成下面样子

struct RoomDetail : View {
    @State var zoomed = false
}

// 展开相当于
struct RoomDetail : View {
    var $zoomed = State<Bool>(initialValue: false)
    var zoomed : Bool {
        get {
            return $zoomed.value
        }
        set {
            $zoomed.value = newValue
        }
    }
}

使用 @State 修饰的状态发生改变，SwiftUI 会再次调用 body, 处理界面的更新。这些具体实现都可以隐藏到 State的 value 读写当中。

手写的代码是不可能定义出 $zoomed这种变量名字的。但在 @propertyDelegate 的实现当中，编译器可以任意修改语法树，插入任意节点，$开头的变量名字让编译器用了。当 @State 修饰了 zoomed，就自动多了名字为 $zoomed，类型为 State的变量。这个变量可以跟 Toggle 之类的 View 绑定。

Toggle(isOn: $zoomed) {
    Text("Favorites only")
}

尾随闭包(Trailing closure)
var body: some View {
    VStack(alignment: .leading, spacing: 10) {
        Text("Hello World")
        Text("Hello SwiftUI")
        Text("Hello Friends")
    }
}

上面的代码应用了有两个语法特性。一个是尾随闭包，另一个是 Function Builders。看 VStack 的定义

public struct VStack<Content> where Content : View {
    @inlinable public init(alignment: HorizontalAlignment = .center, 
                           spacing: Length? = nil, 
                           content: () -> Content)
}

它的 init 函数，最后的参数是个 closure。Swift 的语法中，假如函数最后参数是个 closure，可以提到圆括号外面。上述代码相当于

VStack(alignment: .leading, spacing: 10, content: {
    Text("Hello World")
    Text("Hello SwiftUI")
    Text("Hello Friends")
})

实际调用了 init 函数，生成一个 VStack 对象。另外 Swift 中，假如函数调用中只有一个参数，并且这个参数是个 closure，可以省略函数调用的圆括号。于是

VStack {
    Text("Hello World")
    Text("Hello SwiftUI")
    Text("Hello Friends")
}

实际上也是调用了 VStack 的 init 函数。注意 VStack 的 init 函数，某些参数有默认值，因而某些参数可以省略不写。同理

ForEach(romes)  { rom in
    xxx
}

这种语法也是调用了 ForEach 的 init 函数，生成一个 ForEach 对象，ForEach 也是个 View。

Function Builders
public struct VStack<Content> where Content : View {
    @inlinable public init(alignment: HorizontalAlignment = .center, 
                           spacing: Length? = nil, 
                           content: () -> Content)

VStack 的 init 函数中，closure 需要返回一个 Content。但是下面代码根本就没有返回值，为什么编译成功呢？

VStack {
    Text("Hello World")
    Text("Hello SwiftUI")
    Text("Hello Friends")
}

这里，明明有三个表达式，也不能省略 return。另外加入这样写，会编译错误

// Error
// Closure containing a declaration cannot be used with function builder 'ViewBuilder'
VStack {
    Text("Hello World")
    let a = 1
    Text("Hello SwiftUI")
    Text("Hello Friends")
}

错误信息中出现了 ViewBuilder，是什么东西？

这个语法特性叫 Function Builders[5]，还没有正式添加到 Swift 语言中，只有草案。苹果直接修改了 Swift 编译器，SwiftUI 已经在用这个语法特性了。

VStack 的 init 接口中，其实缺少了@ViewBuilder，将其补充完整，实际是这样。

@_functionBuilder public struct ViewBuilder {
    public static func buildBlock() -> EmptyView
    public static func buildBlock(_ content: Content) -> Content where Content : View
}

public struct VStack<Content> where Content : View {
    public init(..., content: @ViewBuilder () -> Content)

ViewBuilder 结构使用了 @_functionBuilder 来修饰。而闭包使用 @ViewBuilder 来修饰，就会修改语法树，转调 ViewBuilder 的 buildBlock 函数。于是

VStack {
    Text("Hello World")
    Text("Hello SwiftUI")
    Text("Hello Friends")
}

就相当于

VStack {
    return ViewBuilder.builcblock(
        Text("Hello World"), 
        Text("Hello SwiftUI"),
        Text("Hello Friends")
    )
}

Attribute 可以修改语法树，几乎什么神奇的语法都可以实现。就算现存的语法特性不能实现想要的写法，也可以添加新 Attribute，修改 Swift 编译器，让其实现。添加新 Attribute，修改语法树这种大杀器，现在还只能通过修改 Swift 编译器来实现。假如将这大杀器在语法层次暴露出去，让开发者去自定义，几乎就无所不能，可以让 Swift 写得面目全非，但也会引起混乱。

@_functionBuilder 这个语言特性，平时开发逻辑业务是不会用到的，但在写 DSL 就会很有用。比如下面定义，

@_functionBuilder public struct HtmlBuilder {
    public static func buildBlock() -> EmptyView
    public static func buildBlock(_ content: Content) -> Content where Content : View
}


public div(..., content: @HtmlBuilder () -> HtmlNode)
public html(..., content: @HtmlBuilder () -> HtmlNode)
public p(..., content: @HtmlBuilder () -> HtmlNode)
public Text(_ str: String) -> HtmlNode

就可以写出类似的代码

hmtl {
    div {
        Text("Hello World")
        Text("Hello World")

        p {
            Text("Hello World")
        }
        p {
            Text("Hello World")
        }
    }
}

注: 已经有类似的 Html DSL 库了，见Vaux[6]。

附录 1，DSL
DSL 是 Domain Specific Language (领域特定语言) 的缩写。

DSL 是为解决某一个特定任务（领域），专门设计的计算机语言。比如字符串匹配是一个特定任务（有正则表达式），数据库查找是一个特定任务（有 SQL)，都可以设计专门的语言。DSL 选择接近于特定任务的概念，概念甚至直接对应于某个关键字，因而解决特定领域的问题十分高效。但也正因为 DSL 选择特定的概念，当超出了领域范围时，在通用问题上反而表达能力有限。

DSL 通常跟某个宿主语言配合使用。宿主语言集成一个解释器，解释调用 DSL，特定的问题就使用 DSL 来描述。

考虑 DSL 和宿主语言的关系，有两种不同的 DSL。假如 DSL 跟宿主语言是不同的，这个 DSL 就需要专门写一个解释器，称为外部 DSL。SQL、正则表达式都是外部 DSL。而假如 DSL 和宿主语言是相同的语言，有着相同的语法（或者 DSL 语法是宿主语言的子集），这种 DSL 称为叫内部 DSL。

内部 DSL 语法跟宿主语言一样，语法上受到限制。但好处是不用专门去写解释器，跟宿主语言混写，直接使用其编译器（或者解释器），会很方便。通常没有特别注明，说到 DSL 都是指内部 DSL。

有些编程语言，本身语法很灵活（诡异），特别适合于写内部 DSL。SwfitUI 的声明式语法就是 DSL，本身是 Swift 的语法。除了 Swift 语言，比较适合写内部 DSL 的语言还有。

C++, 比如 boost 中的一些库。

Ruby, 比如 CocoaPods 用的 Podfile。

Lua, 比如 bgfx 的接口描述, 见bgfxidl。

Lisp，我不熟悉。

任何语言都可以写 DSL，只是某些语言写起来麻烦一些。比如 Objective-C，语法算很规矩（死板）了，也可以写 Masonry 这种布局库。

附录 2，@dynamicMemberLookup 的实现流程
拿 @dynamicMemberLookup 为例，在 Attr.def 文件有这一行

SIMPLE_DECL_ATTR(dynamicMemberLookup, DynamicMemberLookup,
  OnNominalType,
  9)

SIMPLE_DECL_ATTR 表示 @dynamicMemberLookup 没有参数，不像 @available 那样可以有参数。第二个 DynamicMemberLookup 是名字，会将其解析称 DynamicMemberLookupAttr 成。第三个参数就是表示 Attribute 可标记的位置，OnNominalType 表示在 struct、class、enum、protocol 的前面。9 是编号。

这样加了一行后，就多了一个 Attribute，解析阶段就可以顺利进行，正确解析出节点。参见 ParseDecl.cpp

Parser::parseDecl
Parser::parseDeclAttributeList
Parser::parseDeclAttribute

之后在语义分析阶段，会进行可行性检查，检查不过就编译失败。见TypeCheckAttr.cpp

AttributeChecker::visitDynamicMemberLookupAttr(DynamicMemberLookupAttr *attr)

同理 @available 就会触发 visitAvailableAttr 函数。来到这里，Attribute 的标准处理流程就结束了。不同的 Attribute 可以在任意不同的地方产生作用。具体到 @dynamicMemberLookup，就在 CSSimplify.cpp

ConstraintSystem::performMemberLookup() {
  xxx
  // Recursively look up `subscript(dynamicMember:)` methods in this type.
  auto subscriptName =
    DeclName(ctx, DeclBaseName::createSubscript(), ctx.Id_dynamicMember);
}

修改了语法树节点，创建了一个下标函数节点，将点语法转成了下标函数 subscript(dynamicMember member: String)。这样就用 @dynamicMemberLookup 实现了一个高级的语法糖。

参考
[1]https://xiaozhuanlan.com/topic/7652341890
[2]https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md
[3]https://github.com/apple/swift-evolution/blob/master/proposals/0255-omit-return.md
[4]https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-delegates.md
[5]https://github.com/apple/swift-evolution/blob/9992cf3c11c2d5e0ea20bee98657d93902d5b174/proposals/XXXX-function-builders.md
[6]https://github.com/dokun1/Vaux
