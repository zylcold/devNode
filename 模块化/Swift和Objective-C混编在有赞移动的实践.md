#module #swift 

## 一、概述

随着 Xcode 11、Swift 5.1 的正式发布，Swift 目前已经实现了 ABI 稳定及模块稳定，语法及实现也比以往更加成熟稳定，所以我们在微商城和零售等业务线中尝试使用 Swift 开发部分业务，并在二方库中进行混编开发，在此我们将一些混编经验分享出来。

## 二、现状

同一工程内的混编，通常来讲有两种方式：

1、在宿主工程利用桥接文件（Bridging-Header.h）进行混编

* Swift 访问 Objective-C 只需要在桥接文件中（Bridging-Header.h）中导入需要暴露给 Swift 模块的 Objective-C 类，即可在 Swift 中访问相应 Objective-C 的类和方法
* Objective-C 访问 Swift 在 Objective-C 类中导入 `ProductName-Swift.h` ，即可访问 Swift 中暴露给 Objective-C 的类和方法

2、利用 cocoapods 包管理工具，进行二/三方库混编

* Swift 访问 Objective-C 用 Swift Module 系统，需要用到的 Objective-C 类用 import xxx 进行引用，即可在 Swift 中访问相应的 Objective-C 的类和方法
* Objective-C 访问 Swift 在 Objective-C 类中导入 `ProductName-Swift.h` ，即可访问 Swift 中暴露给 Objective-C 的类和方法

由于我们目前的业务比如商品模块、消息模块、资产模块等都是利用 cocoapods 进行模块化管理，制作成了二方库，供微商城、零售、精选等业务线使用，不建议在宿主工程直接使用 Swift 文件进行业务开发，业务代码应该放到相应的业务模块中去，因此我们将 Swift 代码放入二/三方库中，进行混编。

## 三、Module 系统

### 3.1 LLVM Module 系统

讲到混编方案，就不得不提，苹果在 2012 年 11 月提出 LLVM 的 Module 系统，简单讲就是用树形的结构化的描述来取代以往 `#include` ，例如传统的 `#include` 现在变成 `importstd.io` 。

这样做的主要意义是：

* 语义上完整描述了一个框架的作用
* 提高编译时的可扩展性，同一模块只需编译或导入一次，避免了头文件的多次引用、解析
* 减少碎片化，每个模块只处理一次，环境的变化不会导致不一致

### 3.2 modulemap 文件

`modulemap` 文件就是对一个框架，一个库的所有文件的结构化描述。默认文件名是 `module.modulemap` 关于 LLVM module系统更加详细的内容，可以参考Clang 官方文档

### 3.3 Swift Module

苹果为 Swift 设计了 SwiftModule。SwiftModule 可以将 Swift 解析后生成对应的 modulemap 和 umbrella.h 文件，SwiftModule 增加对编译器版本的依赖，编译产物与编译器 和 Swift 版本有关。如果想要实现 Swift 和 Objective-C 的互相访问，需要 Objective-C 库，以及对应的 umbrella.h 和 modulemap 支持。其中动态库 framework 是 Xcode 支持配置并生成 header，静态库 .a 需要自己编写对应的 umbrella.h 和 modulemap。即库之间无论何种语言实现，均需要封装为 LLVM Module 来相互访问。

LLVM Module 作为苹果公司提出的特性，已经被 Swift 完全采用，在其基础上建立自己的模块系统，当我们结合 Cocoapods 的 use_modular_headers! 配置将三方库构建成静态库，或者 use_frameworks! 配置将三方库构建成动态库时，在编译产物中都会生成一个 modulemap 和 module umbrella.h 文件

可以在 Swift 文件这样引用该模块

### 3.4 use_ modular_ headers！

该特性是 Cocoapods 1.5.0 引入的配置，目的是为了满足 Xcode 9 以后支持的 Swift Static Libraries ，将 Swift Pods 构建成为静态库

* 如果你的 Swift Pod 依赖于 Objective-C，那么你需要为这个 Objective-C 库启用 modular_headers
* 对于 pod 开发者可以在 podtargetxcconfig 内添加 ’DEFINES_MODULE’=>‘YES’，对于使用者在 podfile 内添加 use_modular _headers!
* 在 podspec 中通过 modular_headers => true 配置特定的 pod

可以参考Cocoapods 官方文档

## 四、微商城架构调整

基于上面这些背景，微商城结合团队规模和实践，计划使用壳工程和模块同 git 仓库的 Cocoapods development pod 来替代现有的子项目方式封装模块，模块间依赖基于 podspec 和 podfile 中的配置进行管理。并且为了不中断团队工作和持续交付，实行 Long Term Evolution 长期演进的策略。有关 development pod 可以参考Cocoapods 官方文档。

微商城项目初期：

所有模块均依赖 common 模块，同时所有模块也依赖了 Cocoapods 的二/三方库；在新架构中，common 被封装为 development pod， 并在 podspec 中声明依赖。

调整后，原有的子项目通过头文件暴露的方式仍旧可以访问和依赖，模块间的 Router 和 BeeHive/Bifrost 模块管理也都支持，即该过程对于需求开发团队是无痛的。

最终所有的 development pod 通过 Podfile 集成进壳工程，同时 Podfile 中增加 use _\modular_headers! ，要求 Cocoapods 使用静态库集成并生成对应 modulemap 等 support file。我们在周会上和大家同步了如何将原有的 Xcode 子项目模块迁移到 development pod ，简言之分为三个部分，声明源码，声明资源文件，声明依赖和其他配置，具体 podspec 文档可以参考Cocoapods 官方文档。

最终整体架构如下所示：

在上述版本交付并合并到 master 后，经过完整测试，大家的开发体验没有改变。之后将业务模块也拆分为 development pod ，单个业务模块直接依赖 common pod。在迁移过程中，可以先依赖 common 以实现对二/三方库的依赖。随业务迭代，单业务 development pod 也逐渐理清自身真实的依赖，最终可以把自己的依赖写入 podspec。

## 五、二方库混编

关于二方库混编方案，我们整个有赞移动业务都是用 Cocoapods 来管理二/三方库，声明 use _modular_headers! 将 Swift pods 构建成静态库，目前已经在消息业务模块中已经实践成功，在线上的状况稳定。在此总结了一些混编方案所能遇到的问题。

### 5.1 Framework targets 不支持 Bridging-Header

通常来讲混编的时候需要在工程中创建 Swift 文件时候，Xcode 会问询是否创建 Bridging-Header 文件，点击是，系统会帮你创建一个 Bridging-Header，你可以将需要引用的 Objective-C 模块的头文件放在里面，然后你可以在 Swift 模块用 Objective-C 的类。但是编译器是不允许在 Framework 中创建 Bridging-header，因此在二/三方库中，我们不能使用桥接文件的方式进行混编 Objective-C 代码的引用，需要用 Swift Module 进行模块间的引用。

### 5.2 模块引用

引用其他 Objective-C 二方库需要增加命名空间（Namespace），否则会报错找不到文件 Swift 的命名空间是以模块划分的，一个模块表示一个命名空间。开发时，默认添加到主 target 的内容是同处于同一个命名空间的；如果用 Cocoapods 导入的第三方库，是以一个单独的 target 存在，不会存在命名冲突。但如果以源码的方式导入工程，很可能发生命名冲突，所以为了安全起见，第三方库都会使用命名空间这种方式来防止冲突。

### 5.3 C++ 混编

Objective-C 是 C++ 的超集，就如同 Objective-C 是 C 的超集，在OS X 上同时被 GCC 和 Clang 支持编译，.mm 是 Objective-C++ 的默认后缀名，Xcode 的编译器可以识别。在.mm 文件中，Objective-C 代码和 C++ 代码都可以正常编译运行。在消息业务模块中中引用了 WCDB 这个 Objective-C++ 的库，因此在引用的时候要将引用到的 WCDB.h 头文件中的类文件的 .h 改成 .mm。

### 5.4 链接错误

我们将上述工作做完后引入到宿主工程中，进行编译的时候会出现链接错误，不要担心，那是因为宿主工程中缺少 Swift 的某些系统库，在宿主工程中建立一个 Swift 文件方可解决。

### 5.5 Swift 调用 Objective-C

将 Swift 模块文件中，用import xxx 的形式进行模块的引用，包括 Objective-C 的二/三方库

### 5.6 Objective-C 调用 Swift

* Swift 类中将需要暴露给 Objective-C 模块引用的类，用 public 申明
* Swift 类中需要暴露给 Objective-C 的方法要用关键字 @objc
* 在 Objective-C 类中引用 ProductName-Swift.h 头文件即可引用暴露给 Objective-C 的 Swift 的类和方法

### 5.7 pod spec lint 验证和发布

在 pod spec lint 验证和 pod repo push 发布命令中增加 --use–modular-headers 关键字，否则验证发布不通过

以上是在二方库混编中遇到的一些问题，以供大家参考和探讨。

## 六、优势

* Swift中二进制库的数量逐年攀升，直到iOS13 已经有141个，Foundation 中的许多系统类已经由 Swift 库实现
* ABI 稳定，（iOS12.2系统以上）不增大包体积
* Cocoapods 声明 use_modular_headers! 构建 Swift 静态库，不影响启动速度

## 七、总结

目前微商城项目已经进行了混编项目开发，比如学习中心模块是一个纯 Swift 的二方库，而消息业务模块则是一个 Swift 和 Objective-C 混编的二方库，我们后面会进行越来越多的模块开发用混编的这种形式，新的模块采用 Swift 代码，老的业务还是 Objective-C 不动这种方案。随着 Swift 越来越主流，很多大厂的 App 都用该语言进行开发，但是不能一蹴而就全部将 Objective-C 转成 Swift，而是有很长一段时间都是混编的形式存在，希望该篇文章能够对想进行混编方案的开发者提供一定的参考。

参考文献：

* Swift 官方文档:
[https://swift.org/blog/swift-5-released/](https://swift.org/blog/swift-5-released/)

* Clang 官方文档:
[https://clang.llvm.org/dObjective-Cs/Modules.html](https://clang.llvm.org/dObjective-Cs/Modules.html)

* CocoaPods 官方文档:
[https://guides.cocoapods.org/making/making-a-cocoapod.html](https://guides.cocoapods.org/making/making-a-cocoapod.html)


[Swift和Objective-C混编在有赞移动的实践 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/news/587611)