[原文](https://blog.csdn.net/Hello_Hwc/article/details/85226147)


为了搞清楚这些问题，我们来挖掘下XCode编译iOS应用的细节。

## iOS平台下的编译工具
[[编译器基本知识#编译器]]
Objective C/C/C++
	编译器前端是clang[[编译器基本知识#clang]]
swift
	编译器前端是 [swift](https://swift.org/compiler-stdlib/#compiler-architecture) 

后端都是LLVM [[编译器基本知识#LLVM]] 。

![[image_15.png]]


## [[Clang编译一个C语言文件#例子|一个main.c的例子]]

## XCode编译

通过上文我们大概了解了Clang编译一个C语言文件的过程，但是XCode开发的项目不仅仅包含了代码文件，还包括了图片，plist等。XCode中编译一次都要经过哪些过程呢？

新建一个单页面的Demo工程：CocoaPods依赖AFNetworking和SDWebImage，同时依赖于一个内部Framework。按下Command+B，在XCode的Report Navigator模块中，可以找到编译的详细日志：

![[image_11.png]]

详细的步骤如下：

* 创建Product.app的文件夹
* 把Entitlements.plist写入到DerivedData里，处理打包的时候需要的信息（比如application-identifier）。
* 创建一些辅助文件，比如各种.hmap，这是headermap文件，具体作用下文会讲解。
* 执行CocoaPods的编译前脚本：检查Manifest.lock文件。
* 编译.m文件，生成.o文件。
* 链接动态库，o文件，生成一个mach o格式的可执行文件。
* 编译assets，编译storyboard，链接storyboard
* 拷贝动态库Logger.framework，并且对其签名
* 执行CocoaPods编译后脚本：拷贝CocoaPods Target生成的Framework
* 对Demo.App签名，并验证（validate）
* 生成Product.app

> Tips: Entitlements.plist保存了App需要使用的特殊权限，比如iCloud，远程通知，Siri等。

## 编译顺序

编译的时候有很多的Task(任务)要去执行，XCode如何决定Task的执行顺序呢？

> 答案是：依赖关系。

还是以刚刚的Demo项目为例，整个依赖关系如下：

[image:D032E7BD-F4AA-458A-A814-064A47E09023-1690-000006944CDF9455/image_3.png]
![[image_3.png]]

可以从XCode的Report Navigator看到Target的编译顺序：

[image:91EE9666-9944-4B5A-B1A0-180E56C9B556-1690-000006944CC5C4C0/image_17.png]
![[image_17.png]]

> XCode编译的时候会尽可能的利用多核性能，多Target并发编译。

那么，XCode又从哪里得到了这些依赖关系呢？

* Target Dependencies - 显式声明的依赖关系
* Linked Frameworks and Libraries - 隐式声明的依赖关系
* Build Phase - 定义了编译一个Target的每一步


## 增量编译
日常开发中，一次完整的编译可能要几分钟，甚至几十分钟，而增量编译只需要不到1分钟，为什么增量编译会这么快呢？
因为XCode会对每一个Task生成一个哈希值，只有哈希值改变的时候才会重新编译。
比如，修改了ViewControler.m，只有图中灰色的三个Task会重新执行（这里不考虑build phase脚本）。

[image:739CBE44-924D-486E-98A9-DA984A67F7D8-1690-000006944CA37BE7/image_7.png]
![[image_7.png]]

## 头文件

C语言家族中，头文件(.h)文件用来引入函数/类/宏定义等声明，让开发者更灵活的组织代码，而不必把所有的代码写到一个文件里。

头文件对于编译器来说就是一个promise。头文件里的声明，编译会认为有对应实现，在链接的时候再解决具体实现的位置。

![[image_6.png]]

当只有声明，没有实现的时候，链接器就会报错。

> Undefined symbols for architecture arm64:
> “_umimplementMethod”, referenced from:
> -[ClassA method] in ClassA.o
> ld: symbol(s) not found for architecture arm64
> clang: error: linker command failed with exit code 1 (use -v to see invocation)

Objective C的方法要到运行时才会报错，因为Objective C是一门动态语言，编译器无法确定对应的方法名(SEL)在运行时到底有没有实现(IMP)。

日常开发中，两种常见的头文件引入方式：

```objc
#include "CustomClass.h" //自定义
#include <Foundation/Foundation.h> //系统或者内部framework
```

引入的时候并没有指明文件的具体路径，编译器是如何找到这些头文件的呢？

回到XCode的Report Navigator，找到上一个编译记录，可以看到编译ViewController.m的具体日志：

![[image_8.png]]

把这个日志整体拷贝到命令行中，然后最后加上 `-v` ，表示我们希望得到更多的日志信息，执行这段代码，在日志最后可以看到clang是如何找到头文件的：

```shell
#include "..." search starts here:
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-generated-files.hmap (headermap)
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-project-headers.hmap (headermap)
 /Users/.../Build/Products/Debug-iphoneos/AFNetworking/AFNetworking.framework/Headers
 /Users/.../Build/Products/Debug-iphoneos/SDWebImage/SDWebImage.framework/Headers
 
#include <...> search starts here:
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-own-target-headers.hmap (headermap)
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/Demo-all-non-framework-target-headers.hmap (headermap)
 /Users/.../Build/Intermediates.noindex/Demo.build/Debug-iphoneos/Demo.build/DerivedSources
 /Users/.../Build/Products/Debug-iphoneos (framework directory)
 /Users/.../Build/Products/Debug-iphoneos/AFNetworking (framework directory)
 /Users/.../Build/Products/Debug-iphoneos/SDWebImage (framework directory)
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.0/include
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
 $SDKROOT/usr/include
 $SDKROOT/System/Library/Frameworks (framework directory)
 
End of search list.

```

这里有个文件类型叫做heademap，headermap是帮助编译器找到头文件的辅助文件：存储这头文件到其物理路径的映射关系。

可以通过一个辅助的小工具 [hmap](https://github.com/milend/hmap) 查看hmap中的内容：

```
192:Desktop Leo$ ./hmap print Demo-project-headers.hmap 
AppDelegate.h -> /Users/huangwenchen/Desktop/Demo/Demo/AppDelegate.h
Demo-Bridging-Header.h -> /Users/huangwenchen/Desktop/Demo/Demo/Demo-Bridging-Header.h
Dummy.h -> /Users/huangwenchen/Desktop/Demo/Framework/Dummy.h
Framework.h -> Framework/Framework.h
TestView.h -> /Users/huangwenchen/Desktop/Demo/Demo/View/TestView.h
ViewController.h -> /Users/huangwenchen/Desktop/Demo/Demo/ViewController.h
```

> Tips: 这就是为什么备份/恢复Mac后，需要clean build folder，因为两台mac对应文件的物理位置可能不一样。

clang发现 `#import "TestView.h"` 的时候，先在headermap(Demo-generated-files.hmap,Demo-project-headers.hmap)里查找，如果headermap文件找不到，接着在own target的framework里找：

```
/Users/.../Build/Products/Debug-iphoneos/AFNetworking/AFNetworking.framework/Headers/TestView.h
/Users/.../Build/Products/Debug-iphoneos/SDWebImage/SDWebImage.framework/Headers/TestView.h
```

系统的头文件查找的时候也是优先headermap，headermap查找不到会查找own target framework，最后查找SDK目录。

以 `#import <Foundation/Foundation.h>` 为例，在SDK目录查找时：

首先查找framework是否存在

```shell
$SDKROOT/System/Library/Frameworks/Foundation.framework
```

如果framework存在，再在headers目录里查找头文件是否存在

```shel
$SDKROOT/System/Library/Frameworks/Foundation.framework/headers/Foundation.h
```

## Clang Module
[[Clang Module|Clang Module的简单介绍]]

## Swift编译

现代化的语言几乎都抛弃了头文件，swift也不例外。问题来了，swift没有头文件又是怎么找到声明的呢？

> **编译器干了这些脏活累活** 。编译一个Swift头文件，需要解析module中所有的Swift文件，找到对应的 **声明** 。

![[image_18.png]]

当开发中难免要有Objective C和Swfit相互调用的场景，两种语言在编译的时候查找符号的方式不同，如何一起工作的呢？

**Swift引用Objective C** ：

[Swift的编译器](https://swift.org/compiler-stdlib/) 内部使用了clang，所以swift可以直接使用clang module，从而支持直接import Objective C编写的framework。

![[image_13.png]]

**swift编译器会从objective c头文件里查找符号** ，头文件的来源分为两大类：

* `Bridging-Header.h` 中暴露给swfit的头文件
* framework中公开的头文件，根据编写的语言不通，可能从modulemap或者umbrella header查找
XCode提供了宏定义 `NS_SWIFT_NAME` 来让开发者定义Objective C => Swift的符号映射，可以通过Related Items -> Generate Interface来查看转换后的结果：

![[image_1.png]]

**Objective引用swift**

**xcode会以module为单位，为swift自动生成头文件，供Objective C引用** ，通常这个文件命名为 `ProductName-Swift.h` 。

swift提供了关键词 `@objc` 来把类型暴露给Objective C和Objective C Runtime。

```swift
@objc public class MyClass
```

## 深入理解Linker

> 链接器会把编译器编译生成的多个文件，链接成一个可执行文件。链接并不会产生新的代码，只是在现有代码的基础上做移动和补丁。

链接器的输入可能是以下几种文件：

* object file(.o)，单个源文件的编辑结果，包含了由符号表示的代码和数据。
* 动态库(.dylib)，mach o类型的可执行文件，链接的时候只会绑定符号，动态库会被拷贝到app里，运行时加载
* 静态库(.a)，由ar命令打包的一组.o文件，链接的时候会把具体的代码拷贝到最后的mach-o
* tbd，只包含符号的库文件
这里我们提到了一个概念：符号(Symbols)，那么符号是什么呢？

> 符号是一段代码或者数据的名称，一个符号内部也有可能引用另一个符号。

以一段代码为例，看看链接时究竟发生了什么？

源代码：

```objc
- (void)log{
	printf("hello world\n");
}
```

.o文件：

```
#代码
adrp    x0, l_.str@PAGE
add     x0, x0, l_.str@PAGEOFF
bl      _printf

#字符串符号
l_.str:                                 ; @.str
        .asciz  "hello world\n"
```

在.o文件中，字符串"hello world\n"作为一个符号( `l_.str` )被引用，汇编代码读取的时候按照 `l_.str` 所在的页加上偏移量的方式读取，然后调用printf符号。到这一步，CPU还不知道怎么执行，因为还有两个问题没解决：

1. l_.str在可执行文件的哪个位置？
2. printf函数来自哪里？
再来看看链接之后的mach o文件：

[image:3B93D268-5238-4FCE-A288-4128C19BF5EC-1690-000006944B973F3C/image_9.png]
![[image_9.png]]

链接器如何解决这两个问题呢？

1. 链接后，不再是以 **页+偏移量** 的方式读取字符串，而是直接读虚拟内存中的地址，解决了l_.str的位置问题。
2. 链接后，不再是调用符号_printf，而是在DATA段上创建了一个函数指针 `_printf$ptr` ，初始值为0x0(null)，代码直接调用这个函数指针。启动的时候，dyld会把DATA段上的指针进行动态绑定，绑定到具体虚拟内存中的 `_printf` 地址。更多细节，可以参考我之前的这篇文章： [深入理解iOS App的启动过程](https://blog.csdn.net/Hello_Hwc/article/details/78317863) 。
> Tips: Mach-O有一个区域叫做LINKEDIT，这个区域用来存储启动的时dyld需要动态修复的一些数据：比如刚刚提到的printf在内存中的地址。


## 小结

如有内容错误，欢迎 [issue](https://github.com/LeoMobileDeveloper/Blogs) 指正。

https://img-blog.csdnimg.cn/20181223203254361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvX0h3Yw==,size_16,color_FFFFFF,t_70