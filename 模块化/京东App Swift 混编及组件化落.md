#module #swift

以下文章来源于京东零售技术 
作者彦昌 姚琦 晓峰

## 背景

自 Swift 诞生以来，逐步见证其从饱受诟病到日渐完善。在苹果的全力推动下，潜移默化地把开发支持中心从 Objective-C 转向 Swift，在业界的呼声也越演越烈。当我们相继迎来 ABI稳定、Module stability、Library evolution 等功能后，我们期盼已久的 Swift 已然到来，毅然启动了京东 App 的混编之旅。我们依然坚持稳扎稳打，前期对 Swift 技术做了诸多调研工作，具体可见 [《Swift环境及编译优化调研》](https://mp.weixin.qq.com/s?__biz=MzUyMDAxMjQ3Ng==&amp;mid=2247494257&amp;idx=1&amp;sn=6a62415018b2d14d3f0454ceca1be251&amp;scene=21#wechat_redirect) 。2020年7月京东 App 的首个混编版本上线苹果商店，完成了组件内和主工程的混编工作；近期，我们完成了对京东组件化管理工具（iBiuTool）的改造，混编组件化功能正式落地，这也标志着京东 Swift 混编基础支持建设完毕。但是，Just the beginning...

期待的Swift已经到来

2.1 ABI稳定

Swift 5.0，提供 ABI 稳定，解决了 Swift runtime 的版本兼容问题。这意味着通过 Swift 5.0 及以上的编译器编译出来的二进制，就可以运行在任意 Swift 5.0 及以上的 Swift runtime 上。ABI 稳定后，Swift runtime 和标准库已经植入 macOS 10.14.4、iOS 12.2、watchOS 5.2 及以上系统中。根据苹果官方数据，截止到 2020年12月15日，四年内发布的 iPhone 设备中 iOS 13及以上占比已达 98%。

另外，ABI 稳定还带来了性能上的提升。由于 Swift runtime 已经被深入的集成在了设备的操作系统中，并且结合系统层做了许多优化，这就使得 Swift 程序具有更快的启动速度、更好的运行性能，以及更少的内存占用量。

2.2 Module Stability

Swift 5.1，支持 Module Stability，解决模块间编译器版本兼容的问题。这意味着使用不同版本编译器构建的 Swift 模块可以在同一个应用程序中一起使用。即使某些三方库的 Swift 编译器版本与你所使用的不同，也不会存在编译问题。官方文档中举了一个十分恰当的例子，使用 Swift 6 构建的 framwork，可以被 Swift 6 和未来的 Swift 7 编译器正常使用。所以这个进化对于开发者来说，绝对是一件非常美好事情。

在 Swift 中有一个 .swiftmodule 文件，它是一种二进制文件，主要包含模块中的数据信息和内部编译器的数据结构。由于内部编译器的数据结构的存在，同一个模块编译的 swiftmodule 文件在不同版本的编译器中都是不一样的。这也就是为什么在某个版本编译器中编译的二进制文件，在另一个版本编译器中无法被导入使用的原因。

[image:3E65FB59-D4D0-491F-BA03-064FC85EA8CE-500-00000005C01FD390/640.jpeg]
![[Attachments/640.jpeg]]

Module Stability 解决了这个问题，在模块稳定后，存储模块信息的文件已经替代为 swiftinterface 格式了。它是一个文本格式的文件，它包含所有 public 或者 open 的 API 以及一些隐式的代码或者 API，还包括 swiftinterface 的版本、生成此 swiftinterface 的编译器版本，以及 Swift 编译器将其作为模块导入时所需的命令行标志的子集。而且这些 API 与源代码很类似，通过源码稳定实现了模块稳定。

2.3 Library Evolution

Swift 5.1， 支持 Library Evolution，解决了二进制库向下兼容的问题。在 Library Evolution 特性开启的状态下，二进制库某些场景下的 API 更新后，就会自动实现对旧版本库的兼容。Library Evolution 可以在不破坏二进制兼容性的情况下对库进行某些修改。

举例来具体说明一下这个问题。组件 B 和组件 C 都依赖了组件 A，他们的组件版本都是 v1.0。主工程的 v1.0 发布时，这三个组件需要各种构建，并集成到主工程中。如下图所示：

[image:95576CD8-23C6-47CC-8EDC-37421521D583-500-00000005C01C31DF/_640.jpeg]
![[_640.jpeg]]

当主工程 v2.0 发布时，组件 A 对组件 B 在 v1.0 版本所使用的 API 进行了一些 resilient 的修改，但这些修改并没有影响到组件 C。所以，组件 B 在构建二进制库时，就需要更新依赖的组件 A 到 v2.0 版本。而组件 C 没有功能修改，则不需要更新依赖和发布新版本。然后，他们都集成到 v2.0 版本的主工程中。

[image:0E972F98-0579-4005-B5DE-5E136F40C307-500-00000005C0174E3C/__640.jpeg]
![[__640.jpeg]]

如果组件 A 的 Library Evolution 在没有启用的情况下，在组件 C 中与组件 A 相关的代码就有可能在运行时产生问题、甚至崩溃。而开启 Library Evolution 后，就能够做到对旧版本的兼容。

## 混编的方式

京东 App 根本上是一个基于 Cocoapods 实现的组件化工程，总的来看需要划分为两个场景：一、主工程的混编；二、各组件内的混编。在这两个场景中，对 Swift 引入 ObjC 和 ObjC 引入 Swift又做了不同的处理。

3.1 工程中 - Swift 调用 ObjC

在主工程的 Target 下，需要通过桥接头文件的方式，将 ObjC 的头文件暴露给 Swift 进行使用。

* 创建桥接文件（-Bridging-Header.h）
* 确保 Build Setting 中 SWIFT_OBJC_BRIDGING_HEADER 为该桥接文件的路径
* 将需要引入到 Swift 的 ObjC 的头文件添加进去

3.2 工程中 - ObjC 调用 Swift

在主工程的 Target 下，可以通过引入 Swift Module 的 ObjC Interface Header的方式，在 ObjC 中使用 Swift。由于 Swift Module的缘故，所以引入一个文件，便可以使用该模块下的所有 Swift 文件。需要注意的是，这个头文件的命名默认是"ProjectName-Swift.h"，如果工程名中有一些 nonalphanumeric 字符，则会被替换为下划线。

* 确保 Build Setting 中 SWIFT_OBJC_INTERFACE_HEADER_NAME 的配置正确
* 在 ObjC 中引入该模块的 Swift 头文件，#import "XXX-Swift.h"
* 若在 ObjC的 .h 中引入，则可以通过向前声明的方式，@class XXX

3.3 组件内 - Swift 调用 ObjC

在同一个 .framework 或者 .a 中实现 Swift 调用 ObjC，通过 Bridging-Header 的方式是无法解决的。如果你尝试使用 Bridging-Header 的方式，并且通过 .podspec 对 Bridging-Header 进行配置写入。只会有短暂性的编译成功，最终将会报错：

[image:A48A5B9F-5F99-455D-9CFE-F0348CF1DD80-500-00000005C012EF2F/___640.jpeg]

![[___640.jpeg]]

经过官方文档中的近一步查证，发现在同一个 framework 中的 Swift 想要引入 ObjC，需要将该 ObjC 文件导入到其 umbrella-header 文件中。这样 Swift 模块就可以对 umbrella-header 中向外暴露的类进行调用了。另外，官方文档中还提到 DEFINES_MODULE 要配置为 YES，这样整个组件就可以作为一个模块被外部导入使用了。

3.4 组件内 - ObjC 调用 Swift

在同一个 .framework 或者 .a 中实现 ObjC 调用 Swift，依然需要通过引入 Swift Module 的 ObjC Interface Header。

* 确保 Build Setting 中 SWIFT_OBJC_INTERFACE_HEADER_NAME 的配置正确
* 在 ObjC 中引入该模块的 Swift 头文件，.framework 中为 `#import <XXX/XXX-Swift.h>`，.a 中为 `#import "XXX-Swift.h"`。
* 若在 ObjC的 .h 中引入，则可以通过向前声明的方式，@class XXX

组件内混编

4.1 组件内混编实施方案

参照上文中梳理的大体方案，便可以对京东App组件进行混编实施。京东组件通过自己的工具进行组件管理的，由于历史原因，某些方面的功能还不能完全支持，比如 modulemap、module stability、new build setting。所以这一些问题需要绕过，这些问题文章后面会针对说明。具体的实施步骤如下：

* 组件内添加 Swift 文件，且不需要创建桥接文件
* podspec 中 source_files 配置中添加 swift 项
* ObjC 调用 Swift

o 在苹果官方文档中，推荐配置 DEFINES_MODULE = YES，并通过 `#import <XXX/XXX-Swift.h> `的方式导入 Swift Module。但在京东组件中动态库是以 .framework 形式存在的，需要以 `#import <XXX/XXX-Swift.h> `的方式导入 Swift Module；而静态库是以 .a 形式存在的，需要以 `#import "XXX-Swift.h"` 的方式导入。

o 值得注意的是，要确保你的 Swift 类为 Public 或 Open 的访问权限，否则在 Swift Module 之外的 ObjC 文件中是无论如何都不能调用的。对于 ObjC 中想要使用属性和函数，需要标记 @objc，它会告诉编译器该属性或者函数能够应用于 Objective-C 代码中。而且标有 @objc 特性的类必须继承自 ObjC 的类。

* Swift 调用 ObjC

o Swift 模块想要调用 ObjC 就需要将 ObjC 的头文件暴露在 umbrella-header 中。这样就需要在 podspec 中将 public_header_files 配置中添加要暴露的 ObjC 头文件后，供 Swift 进行调用。

4.2 组件内混编通信方案

按照上述方案实施后，组件内的通信归为 ObjC 调用 Swift 和 Swift 调用 ObjC 两个方面。具体通信方式如下图所示：

[image:FA595168-02D8-4A9C-AE66-1575B8E13A74-500-00000005C00DE854/
![[____640.jpeg]]


## 组件间混编

5.1 Swift 调用 ObjC

Swift 调用 ObjC API 前，首先需要通过 import module 语法找到对应的模。Module机制是在2013年加入了Xcode中，目的是为了提升编译速度，解决C、C++中#include机制的一些遗留问题。具体是如何解决的，可以看下这两篇文章：关于objective-cmodules和autolinking（1）、Clang官方文档（2）。

我们需要解决的问题是如何让编译器找到Module，通过查看 Clang 官方的文档，我们发现：

* 如果要支持Module，必须提供一个module.modulemap文件，用来声明模块与头文件之间的映射关系
* 针对 framework，Clang 会通过指定路径查找命名为module.modulemap的文件：.framework/Modules/module.modulemap

module.modulemap文件中的内容大致如下，主要是用来声明模块与头文件之间的映射关系，支持 import module 方式调用。

```swift
framework module STStaticBasicStableModule {       
	umbrella header "STStaticBasicStableModule-umbrella.h"       
	export *       
	module * {  
		export * 
	}   
}   
module STStaticBasicStableModule.Swift {       
	header "STStaticBasicStableModule-Swift.h"       
	requires objc   
}
```

找到模块，并且知道模块有哪些头文件后，就可以访问组件提供的类、方法了。

5.2 Swift 调用 Swift

我们知道 ObjC 代码之间调用 API 是通过头文件的形式，但 Swift 是没有头文件的，它使用一个二进制格式的文件（.swiftmodule）来代替头文件，这个文件中包含了 Swift 模块的所有 API、inlinable function bodies。

编译器会去哪找 swiftmodule 文件呢？我们在 swift 源码中找到了一些蛛丝马迹：

```swift
// SerializedModuleLoader.cpp   
void SerializedModuleLoaderBase::collectVisibleTopLevelModuleNamesImpl(       SmallVec torImpl<Identifier> &names, StringRef extension) const {     
	// ...     
	forEachModuleSearchPath(Ctx, [&](StringRef searchPath, SearchPathKind Kind, bool isSystem) {       
		switch (Kind) {       
			// ...       
			case SearchPathKind::Framework: {         
				// 源码中的注释及相关代码说明了 swiftmodule 在 framework 中的查找机制         // Look for:         
				// $PATH/{name}.framework/Modules/{name}.swiftmodule/{arch}.{extension}         
				forEachDirectoryEntryPath(searchPath, [&](StringRef path) {           // ...         
				});         
				return None;       
			}       
		}       
		llvm_unreachable("covered switch");     
	});   
}
```

.swiftmodule是一个序列化后的二进制文件，从文件名SerializedModuleLoader.cpp可以猜测这个是负责加载.swiftmodule文件的。另外上述代码中也说明了针对 framework，Swift 编译器如何查找.swiftmodule。

同时为了保证组件支持 x86、arm 架构下编译，我们还需要将不同架构编译生成的 swiftmodule 文件手动合并到最终的 framework 中。

5.3 ObjC 调用 Swift

编译器会通过我们编写的 Swift 代码生成xxx-Swift.h，这样 ObjC 就可以通过这个头文件访问 Swift 的 API 了。我们可以通过两种 import 方式调用 Swift API：

* `@import STStaticBasicStableModule`
* `#import "STStaticBasicStableModule-Swift.h"`

还记得上面 modulemap 中的STStaticBasicStableModule.Swift吧，@import STStaticBasicStableModule在找到模块后，会通过 modulemap 文件中声明找到STStaticBasicStableModule-Swift.h：

```swift
module STStaticBasicStableModule.Swift {       
	header "STStaticBasicStableModule-Swift.h"       
	requires objc   
}
```

xxx-Swift.h也需要支持多架构，我们需要把不同架构下生成的xxx-Swift.h内容合并到一个文件中，最终合并的xxx-Swift.h，去掉部分代码，大致结构是这个样子：

```objc
#ifndef TARGET_OS_SIMULATOR   
#include <TargetConditionals.h>   
#endif       
#if TARGET_OS_SIMULATOR           
// Release-iphonesimulator/Swift Compatibility Header/XXX-Swift.h       
#if 0       
#elif defined(__x86_64__) && __x86_64__       
// __x86_64__       
#elif defined(__i386__) && __i386__       
// __i386__       
#endif       
#else           
// Release-iphoneos/Swift Compatibility Header/XXX-Swift.h       
#if 0       
#elif defined(__arm64__) && __arm64__       
// __arm64__       
#elif defined(__ARM_ARCH_7A__) && __ARM_ARCH_7A__       
// __ARM_ARCH_7A__       
#endif       
#endif 
// TARGET_OS_SIMULATOR
```

5.4Module stability & Library evolution

文章开篇说过，它们是 Swift 5.1 新增的2个关于二进制稳定的特性，可以支持发布和共享 framework。只有当你的库要独立于客户端进行构建的情况下，才需要开启 Build Libraries for Distribution 选项，而且 Module Stability 和 Library Evolution 就会同时生效。通常我们在开发二进制库时，最好尽早打开此开关，以提供任何二进制兼容性的保证。

如果不支持这2个特性，可能会出现的问题：

* User1 使用 Xcode 11.2 发布了基础组件，User2 依赖了这个组件，并使用 Xcode 11.7 编译，结果：编译报错Module compiled with Swift 5.1.2 cannot be imported by the Swift 5.2.4compiler
* User1 给结构体中新增了一个属性，然后发布了新版本组件，依赖该组件的下游较多，User1 需要周知所有依赖方依赖最新版本重新编译，否则可能会引发运行时崩溃

开启这2个特性的方式：

* Xcode 中设置 build setting 中的 BUILD_LIBRARY_FOR_DISTRIBUTION 为 YES
* 需要支持 New Build System。

5.5组件间混编通信方案

按照上述方案实施后，组件间的通信归为 ObjC 调用 Swift 和 Swift 调用 ObjC，以及 Swift 调用 Swift 三个方面。具体通信方式如下图所示：

[image:00C9DED1-E722-4FC6-9829-84BBD978335D-500-00000005C009EE42/_____640.jpeg]
![[_____640.jpeg]]

京东主工程混编支持方案

6.1静态编译的问题

由于我们之前是纯 ObjC 的开发环境，所以即使实现了组件内混编，京东组件化的主工程（或者组件的Example工程）也并不能成功编译。原因在于它们还不支持 Swift 混编环境，在编译时可能会报类似错误：

[image:E7054EF5-75C6-4E8A-9185-E7767063C57D-500-00000005C0060723/______640.jpeg]
![[______640.jpeg]]

或者

[image:7CC71A5A-C4E9-42E3-A132-ED8CA29C4A4D-500-00000005C0012067/_______640.jpeg]
![[_______640.jpeg]]

上述的错误信息说明，编译器不能自动链接到 Swift 相关的一些静态库和动态库。而这些资源是存在于 Xcode Toolchains 下的，在本地的路径为：

* /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/iphoneos
* /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift-5.0/iphoneos

接下来将这些资源路径配置在工程中，一般地需要在Build Settings -> Library Search Paths 中添加：

* "$(TOOLCHAIN_DIR)/usr/lib/swift/$(PLATFORM_NAME)"
* "$(TOOLCHAIN_DIR)/usr/lib/swift-5.0/$(PLATFORM_NAME)"

6.2动态库加载的问题

当配置好可链接资源的路径之后，就可以成功编译了。但在启动时，动态库加载的问题将会引起程序的崩溃。诸如以下错误：

[image:9F50A8F8-9233-4593-9EE4-7152E4F603D8-500-00000005BFFD023F/________640.jpeg]
![[________640.jpeg]]

如果你的设备是 iOS 12.2 及以上，可能报错如下：

[image:E4A82767-3778-4BB9-B868-E447FA63105C-500-00000005BFF8F7C0/_________640.jpeg]
![[_________640.jpeg]]

在 Build Settings -> Runpath Search Paths 中首行添加配置 /usr/lib/swift 配置，特别要注意的是只能在首行配置才能解决问题。

如果你的设备是 iOS 12.2 以下，可能报错如下：

[image:BC1C9202-58F2-47D1-BD0B-4A22DF62FC2A-500-00000005BFF4CCD6/__________640.jpeg]
![[__________640.jpeg]]

iOS 12.2 以下，将 Build Settings -> Always Embed Swift Standard Libraries 设置为 YES。

这两个问题都是在应用启动后，动态库加载时，发生的崩溃。那为什么要以 iOS 12.2 为分水岭呢？这就是从 iOS 12.2 Swift 已经实现 ABI 稳定了。所以上述两处的解决方案缺一不可，因为它们分别针对 ABI 稳定前后的系统版本。这些配置完成后，京东组件的主工程（或者组件的Example工程）就已经完全支持 Swift 混编环境了。另外，还有一种更加便捷方案也可以达到同样的效果。

6.3一键配置混编环境

除了 Xcode Build Settings 配置的方式，还可以通过在工程中新建一个 Swift 文件（文件中无需添加任何代码），通过这种方式 Xcode 会自动完成部分环境配置，能够解决静态编译问题和部分设备的动态库加载问题。另外，还需要处理 iOS 12.2 以下 Swift 动态库加载的问题。将 Always Embed Swift Standard Libraries 设置为 YES。

假如通过 Xcode Build Settings 配置的方式，从 Xcode 11 beta 4 开始，就需要在工程配置中添加 swift-5.0 的新配置项了。但如果通过新建 Swift 文件的方式，就不必更新配置，它的优势在于机动性好。

京东是一个标准的组件化应用，主工程中无代码实现，所以不需要实现混编代码，仅需要使其支持 Swift 开发环境即可。因此 Xcode Build Settings 配置的方式能够满足需求，无需多余文件，在工程的简洁性上更好。

通信方案总结

最后，对京东 App 中涵盖的混编通信方式做个汇总。以组件内、组件间，以及主工程的混编形式为基础，将整体的混编通信方式汇总如下：

[image:22A9FAAA-5C5E-4C4B-983B-03138A404CA6-500-00000005BFF05362/___________640.jpeg]
![[___________640.jpeg]]

作者:王彦昌、姚琦、林晓峰


参考文献

* https://onevcat.com/2013/06/new-in-xcode5-and-objc/#%E5%85%B3%E4%BA%8Eobjective-cmodules%E5%92%8Cautolinking
* https://clang.llvm.org/docs/Modules.html
* https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_objective-c_into_swift
* https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_swift_into_objective-c
* http://clang.llvm.org/docs/Modules.html
* https://swift.org/blog/library-evolution/
* https://swift.org/blog/abi-stability-and-more/
* https://forums.swift.org/t/plan-for-module-stability/14551
* https://forums.swift.org/t/learning-about-swifts-lack-of-header-files/13215/5
* https://github.com/apple/swift
* https://github.com/apple/swift-evolution/blob/master/proposals/0260-library-evolution.md