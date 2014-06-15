---
layout: post
title: "Swift and C Interop(简析Swift和C的交互)"
date: 2014-06-15 00:11:44 +0800
comments: true
categories: swift
---

之前好像简单说过 Swift 和 Objective-C 的交互问题。

其实也许有种可能是我们用 Swift 调用纯 C 代码。这里不会也永远不会考虑 C++ 的情况，因为不支持，不过可以用 C 写 wrapper, 这个没有任何问题。

Swift 官方文档中，以及那本已经被迅速翻译为中文的 ibooks 书中，都提到了 Swift 调用 Objective-C 和 C 是有很好支持的。不过没有细节。

这里不会考虑如何用 C 调用 Swift, 暂时，看心情。：）

这里主要面向 MacOSX 应用。iOS 或许可以适用。

先复习下区别。

# 第一部分 预备知识

## 语言区别

对于 C 来说，最头疼的莫过于指针，而 Swift 是一门没有指针的语言。是不是要吐血？相对来说指针不但代表者指针操作传参，还有指针运算等等。

# 第二部分 调用 C

## C 标准库

好消息是，对于标准库中的 C 函数，根本不需要考虑太多导入头文件神马的。比如 ``strlen``、``putchar``、``vprintf``。当然 ``vprintf`` 需要多说几句，后面说。

请直接 ``import Darwin`` 模块。

这些标准库函数表示为 ``Darwin.C.HEADER.name``。

实际上由于 Swift 模块结构是平坦的，他们均位于 ``Darwin`` 中，所以基本上是直接用的。

然后 ``CoreFoundation`` 用到了 ``Darwin`` ( @exported 导入，所以相当于这些名字也在 ``CoreFoundation`` 中)。

然后 ``Foundation`` 用到了 ``CoreFoundation`` （也是 @exported 导入。）

所以其实你导入 ``Foundation`` 的时候，这些 C 函数都是直接可用的。比如 ``putchar`` 一类。

多说一句，``Cocoa`` 当然也包含 ``Foundation``。

## C 函数

好吧假设你有个牛逼到顶天的算法是 C 写的，不对，假设是别人写的，牛逼到你只能凑合用却看不懂然后自己也写不出没空迁移的地步。

我们直接创建一个 Swift 项目，然后 New File，添加一个 ``.c`` 文件。

这时候 Xcode 会弹出对话框询问是否配置 Bridge Header，确认就可以了。也可以手动添加 Bridge Header，位置在项目的 Build Settings 中的 Swift Compiler - Code Generation 子项里。指向你的 Bridge Header 文件名就可以了。

一般这个文件是 ``ProjectName-Bridging-Header.h``。情况基本和与 Objective-C 混编没区别。

剩下的工作就很简单了。在 ``.c`` 文件填上传说中的牛逼算法。在 ``ProjectName-Bridging-Header.h`` 中加上该函数原型或者引入相关的头文件。

在 Swift 中调用的名字和 C 名字一样就可以了，比如你定义了一个 ``int mycsort()`` 那么在 Swift 中就是 ``func mycsort() -> CInt``。

这时候问题来了。一个漂亮的问题。

我的 C 函数名字和 Swift 标准库冲突怎么办？比如我定义了一个函数就叫 ``println``，我们知道 Swift 里也有个 ``println``。

这样，如果直接调用会提示 ``Ambiguous use of 'println'``。没辙了么？

这里有个我发现的 Undocumented Featuer 或者说是 Undocumented Attribute。你转载好歹提下我好吧。（发现方法是通过 Xcode 查看定义，然后通过 nm 命令发现符号, 对照 llvm ir 确认的。)

那就是 ``@asmname("func_name_in_c")``。用于函数声明前。使用方法：

```c
int println() { .... }
```

```
@asmname("println") func c_println() -> CInt // 声明，不需要 {} 函数体
c_println() // 调用
```

也就是 C 中的同名函数，我们可以给赋予一个别名，然后正常调用。这么一看就基本没有问题了。至于类型问题，待会说，详细说。

## C Framework

很多时候我们拿到的是第三方库，格式大概是个 Framework。比如 ``SDL2.framework``。举这个例子是因为我想对来说比较熟悉 SDL2。

直接用 Finder 找到这个 ``.framework`` 文件，拖动到当前项目的文件列表里，这样它将作为一个可以展开的文件夹样式存在于我们的项目中。

在 ``ProjectName-Bridging-Header.h`` 中引入其中需要的 ``.h``。

比如我们引入 ``SDL2.framework``，那么我们就需要写上 ``#import <SDL2/SDL.h>``。

然后在 Swift 文件里正常调用就好了。

所以其实说到底核心就是那个 ``ProjectName-Bridging-Header.h``，因为它是作为参数传递给 Swift 编译器的，所以 Swit 文件里可以从它找到定义的符号。

但是，这个桥接头文件的一切都是隐式的，类型自动对应，所以很多时候需要我们在 Swift 里调用并封装。或者使用 ``@asmname(...)`` 避免名字冲突。

# 第三部分 类型转换

前面说到了 C 中有指针，而 Swift 中没有，同时基本类型还有很多不同。所以混编难免需要在两种语言的不同类型之间进行转换。

牢记一个万能函数 ``reinterpretCast<T, U>(T) -> U``，只要 T, U sizeof 运算相等就可以直接转换。这个在之前的标准库函数里有提到。调用 C 代码的利器！

## 基本类型对应

- int => CInt
- char => CChar / CSignedChar
- char* => CString
- unsigned long = > CUnsignedLong
- wchar_t => CWideChar
- double => CDouble
- T* => CMutablePointer<T>
- void* => CMutableVoidPointer
- const T* => CConstPointer<T>
- const void* => CConstVoidPointer
- ...

继续这个列表，你肯定会想这么多数值类型，怎么搞。其实都是被 ``typealias`` 定义到 ``UInt8``，``Double`` 这些的。放心。C 中数值类型全部被明确地用别名定义到带 size 的 Swift 数值类型上。完全是一样用的。

其实真正的 Pointer 类型只是 ``UnsafePointer<T>``，大小与 C 保证一致，而对于这里不同类型的 Pointer，其实都是 ``UnsafePointer``
到它们的隐式类型转换。还有个指针相关类型是 ``COpaquePointer``.

同时 ``NilType``，也就是 ``nil`` 有到这些指针的隐式类型转换。所以可以当做 ``NULL`` 用。

还有个需要提到的类型是 CString, 他的内存 layout 等于 ``UnsafePointer<UInt8>``，下面说。

## CString 

用于表示 ``char *``，``\0`` 结尾的 c 字符串，实际上似乎还看到了判断是否 ASCII 的选项，但是没试出来用法。

实现了 ``StringLiteralConvertible`` 和 ``LogicValue``。目前看 ``LogicValue`` 实现有 BUG。从字符串常量直接赋值是没有问题的。



和 String 的转换通过一个 extension 实现，也是很方便。

```scala
extension String {
  static func fromCString(cs: CString) -> String
  static func fromCString(up: UnsafePointer<CChar>) -> String
}
// 还有两个方便的函数。 Rust 背景的同学一定仰天长啸。太相似了。
extension String {
  func withCString<Result>(f: (CString) -> Result) -> Result
  func withCString<Result>(f: (UnsafePointer<CChar>) -> Result) -> Result
}

```

``char *`` 的类型会对应为 ``UnsafePointer<CChar>``，而实际上 ``CString`` 更适合。所以在 Swift 代码中，往往我们要再次申明下这个函数。或者用一个函数分装下，转换成我们需要的类型。

例如，假设在 Bridging Header 中我们声明了 ``char * foo();``，那么，在 Swift 代码中我们可以用上面提到的方法：

```
@asmname("foo") func c_foo() -> CString
// 注意这里没有 {}
let ret = c_foo()
```

当然也可以直接调用原始函数然后转换。不过由于 ``char`` 对应 ``CChar``，而实际上是 ``Int8`` 的别名，而 ``CString`` 转换需要的类型是 ``UnsafePointer<UInt8>``，崩溃了吧，实际上在很多其他语言也是这样，所以建议可以修改下 Header 的声明类型，这个是不会影响的。改为 ``unsigned char * foo()``

```scala
let raw = foo() // UnsafePointer<UInt8>
let ret = CString(raw) // CString
let str = String.fromCString(ret)
```

这么用还是略郁闷的，所以可以用 ``reinterpretCast()``，直接转换。但是请一定要知道自己在转换什么，确保类型的 sizeof 相同，确保转换本身有意义。

## Unmanaged

这个先挖坑，随后填上。
