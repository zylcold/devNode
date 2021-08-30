[原文](https://blog.csdn.net/Hello_Hwc/article/details/85226147)

## 前言

两年前曾经写过一篇关于编译的文章《 [iOS编译过程的原理和应用](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/iOS%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E7%9A%84%E5%8E%9F%E7%90%86%E5%92%8C%E5%BA%94%E7%94%A8.md) 》，这篇文章介绍了iOS编译相关基础知识和简单应用，但也很有多问题都没有解释清楚：

* Clang和LLVM究竟是什么
* 源文件到机器码的细节
* Linker做了哪些工作
* 编译顺序如何确定
* 头文件是什么？XCode是如何找到头文件的？
* Clang Module
* [[理解iOS签名]]


为了搞清楚这些问题，我们来挖掘下XCode编译iOS应用的细节。

## [[编译器相关#编译器]]
Objective C/C/C++
	编译器前端是clang[[编译器相关#clang]]
swift
	编译器前端是 [swift](https://swift.org/compiler-stdlib/#compiler-architecture) 

后端都是LLVM [[编译器相关#LLVM]] 。

![[image_15.png]]
## [[C构建流程#过程]]

接下来，从代码层面看一下具体的转化过程，新建一个main.c:

```c
#include <stdio.h>
// 一点注释
#define DEBUG 1
int main() {
#ifdef DEBUG
  printf("hello debug\n");
#else
  printf("hello world\n");
#endif
  return 0;
}
```

## [[C构建流程#预处理]]

`#include "stdio.h"` 就是告诉预处理器将这一行替换成头文件 `stdio.h` 中的内容，这个过程是递归的：因为 `stdio.h` 也有可能包含其头文件。

用clang查看预处理的结果：

```shell
xcrun clang -E main.c
```

预处理后的文件有400多行，在文件的末尾，可以找到main函数

```shell
int main() {
  printf("hello debug\n");
  return 0;
}
```

可以看到，在预处理的时候，注释被删除，条件编译被处理。


## [[C构建流程#词法分析]]

```shell
$ xcrun clang -fmodules -fsyntax-only -Xclang -dump-tokens main.c
```

输出：

```shell
annot_module_include '#include <s'		Loc=<main.c:1:1>
int 'int'	 [StartOfLine]	Loc=<main.c:4:1>
identifier 'main'	 [LeadingSpace]	Loc=<main.c:4:5>
....

```

`Loc=<main.c:1:1>` 标示这个token位于源文件main.c的第1行，从第1个字符开始。保存token在源文件中的位置是方便后续clang分析的时候能够找到出错的原始位置。

## [[C构建流程#语法分析]]

```shell
$ xcrun clang -fsyntax-only -Xclang -ast-dump main.c | open -f
```

main函数AST的结构如下：

```shell
[0;34m`-[0m[0;1;32mFunctionDecl[0m[0;33m 0x7fcc188dc700[0m <[0;33mmain.c:4:1[0m, [0;33mline:11:1[0m> [0;33mline:4:5[0m[0;1;36m main[0m [0;32m'int ()'[0m
[0;34m  `-[0m[0;1;35mCompoundStmt[0m[0;33m 0x7fcc188dc918[0m <[0;33mcol:12[0m, [0;33mline:11:1[0m>
[0;34m    |-[0m[0;1;35mCallExpr[0m[0;33m 0x7fcc188dc880[0m <[0;33mline:6:3[0m, [0;33mcol:25[0m> [0;32m'int'[0m[0;36m[0m[0;36m[0m
[0;34m    | |-[0m[0;1;35mImplicitCastExpr[0m[0;33m 0x7fcc188dc868[0m <[0;33mcol:3[0m> [0;32m'int (*)(const char *, ...)'[0m[0;36m[0m[0;36m[0m <[0;31mFunctionToPointerDecay[0m>
[0;34m    | | `-[0m[0;1;35mDeclRefExpr[0m[0;33m 0x7fcc188dc7a0[0m <[0;33mcol:3[0m> [0;32m'int (const char *, ...)'[0m[0;36m[0m[0;36m[0m [0;1;32mFunction[0m[0;33m 0x7fcc188c5160[0m[0;1;36m 'printf'[0m [0;32m'int (const char *, ...)'[0m
[0;34m    | `-[0m[0;1;35mImplicitCastExpr[0m[0;33m 0x7fcc188dc8c8[0m <[0;33mcol:10[0m> [0;32m'const char *'[0m[0;36m[0m[0;36m[0m <[0;31mBitCast[0m>
[0;34m    |   `-[0m[0;1;35mImplicitCastExpr[0m[0;33m 0x7fcc188dc8b0[0m <[0;33mcol:10[0m> [0;32m'char *'[0m[0;36m[0m[0;36m[0m <[0;31mArrayToPointerDecay[0m>
[0;34m    |     `-[0m[0;1;35mStringLiteral[0m[0;33m 0x7fcc188dc808[0m <[0;33mcol:10[0m> [0;32m'char [13]'[0m[0;36m lvalue[0m[0;36m[0m[0;1;36m "hello debug\n"[0m
[0;34m    `-[0m[0;1;35mReturnStmt[0m[0;33m 0x7fcc188dc900[0m <[0;33mline:10:3[0m, [0;33mcol:10[0m>
[0;34m      `-[0m[0;1;35mIntegerLiteral[0m[0;33m 0x7fcc188dc8e0[0m <[0;33mcol:10[0m> [0;32m'int'[0m[0;36m[0m[0;36m[0m[0;1;36m 0[0m
```

## [[C构建流程#CodeGen]]

```shell
xcrun clang -S -emit-llvm main.c -o main.ll
```

main.ll文件内容：

```shell
...
@.str = private unnamed_addr constant [13 x i8] c"hello debug\0A\00", align 1

; Function Attrs: noinline nounwind optnone ssp uwtable
define i32 @main() #0 {
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([13 x i8], [13 x i8]* @.str, i32 0, i32 0))
  ret i32 0
}
...

```


## [[C构建流程#生成汇编代码]]

生成arm 64汇编：

```shell
$ xcrun clang -S main.c -o main.s
```

查看生成的main.s文件，篇幅有限，对汇编感兴趣的同学可以看看我的这篇文章： [iOS汇编快速入门](https://github.com/LeoMobileDeveloper/Blogs/blob/master/Basic/iOS%20assembly%20toturial%20part%201.md) 。

```shell
_main:                                  ## @main
        .cfi_startproc
## %bb.0:
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
...

```

## [[C构建流程#汇编器]]

```shell
$ xcrun clang -fmodules -c main.c -o main.o
```

还记得我们代码中调用了一个函数 `printf` 么？通过nm命令，查看下main.o中的符号

```shell
$ xcrun nm -nm main.o
                 (undefined) external _printf
0000000000000000 (__TEXT,__text) external _main
```

`_printf` 是一个是undefined external的。undefined表示在当前文件暂时找不到符号 `_printf` ，而external表示这个符号是外部可以访问的，对应表示文件私有的符号是 `non-external` 。

**Tips** ：什么是符号(Symbols)? 符号就是指向一段代码或者数据的名称。还有一种叫做WeakSymols，也就是并不一定会存在的符号，需要在运行时决定。比如iOS 12特有的API，在iOS11上就没有。

## [[C构建流程#链接]]

```shell
$ xcrun clang main.o -o main
```

我们就得到了一个mach o格式的可执行文件

```shell
$ file main
main: Mach-O 64-bit executable x86_64
$ ./main 
hello debug
```

在用nm命令，查看可执行文件的符号表：

```shell
$ nm -nm main
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000f60 (__TEXT,__text) external _main
```

_printf仍然是 `undefined` ，但是后面多了一些信息： `from libSystem` ，表示这个符号来自于 `libSystem` ，会在运行时动态绑定。


## XCode编译

通过上文我们大概了解了Clang编译一个C语言文件的过程，但是XCode开发的项目不仅仅包含了代码文件，还包括了图片，plist等。XCode中编译一次都要经过哪些过程呢？

新建一个单页面的Demo工程：CocoaPods依赖AFNetworking和SDWebImage，同时依赖于一个内部Framework。按下Command+B，在XCode的Report Navigator模块中，可以找到编译的详细日志：

[image:6E415C94-2AB7-42A3-A786-4ADE848929B3-1690-000006944CFA8B2F/image_11.png]
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

[image:4E33AFB2-F1F8-4748-86A3-BF8DBD9603CE-1690-000006944C83651E/image_6.png]
![[image_6.png]]

当只有声明，没有实现的时候，链接器就会报错。

> Undefined symbols for architecture arm64:
> “_umimplementMethod”, referenced from:
> -[ClassA method] in ClassA.o
> ld: symbol(s) not found for architecture arm64
> clang: error: linker command failed with exit code 1 (use -v to see invocation)

Objective C的方法要到运行时才会报错，因为Objective C是一门动态语言，编译器无法确定对应的方法名(SEL)在运行时到底有没有实现(IMP)。

日常开发中，两种常见的头文件引入方式：

```
#include "CustomClass.h" //自定义
#include <Foundation/Foundation.h> //系统或者内部framework
12
```

引入的时候并没有指明文件的具体路径，编译器是如何找到这些头文件的呢？

回到XCode的Report Navigator，找到上一个编译记录，可以看到编译ViewController.m的具体日志：

[image:D74DA21E-96FC-40E8-B1D0-E62C9BED2826-1690-000006944C6A3159/image_8.png]
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
1234567
```

> Tips: 这就是为什么备份/恢复Mac后，需要clean build folder，因为两台mac对应文件的物理位置可能不一样。

clang发现 `#import "TestView.h"` 的时候，先在headermap(Demo-generated-files.hmap,Demo-project-headers.hmap)里查找，如果headermap文件找不到，接着在own target的framework里找：

```
/Users/.../Build/Products/Debug-iphoneos/AFNetworking/AFNetworking.framework/Headers/TestView.h
/Users/.../Build/Products/Debug-iphoneos/SDWebImage/SDWebImage.framework/Headers/TestView.h
12
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

传统的 `#include/#import` 都是文本语义：预处理器在处理的时候会把这一行替换成对应头文件的文本，这种简单粗暴替换是有很多问题的：

1. 大量的预处理消耗。假如有N个头文件，每个头文件又 `#include` 了M个头文件，那么整个预处理的消耗是 **N*M** 。
2. 文件导入后，宏定义容易出现问题。因为是文本导入，并且按照include依次替换，当一个头文件定义了 `#define std hello_world` ，而第另一个个头文件刚好又是C++标准库，那么 `include` 顺序不通，可能会导致所有的std都会被替换。
3. 边界不明显。拿到一组.a和.h文件，很难确定.h是属于哪个.a的，需要以什么样的顺序导入才能正确编译。
[clang module](https://clang.llvm.org/docs/Modules.html) 不再使用文本模型，而是采用更高效的语义模型。clang module提供了一种新的导入方式:`@import` ，module会被作为一个独立的模块编译，并且产生独立的缓存，从而大幅度提高预处理效率，这样时间消耗从 **M*N** 变成了 **M+N** 。

XCode创建的Target是Framework的时候，默认define module会设置为YES，从而支持module，当然像Foundation等系统的framwork同样支持module。

`#import <Foundation/NSString.h>` 的时候，编译器会检查 `NSString.h` 是否在一个module里，如果是的话，这一行会被替换成 `@import Foundation` 。

[image:D9E4ADED-512E-4BF2-8E00-816B88C3E606-1690-000006944C3A8CD9/image_19.png]
![[image_19.png]]


那么，如何定义一个module呢？答案是：modulemap文件，这个文件描述了一组头文件如何转换为一个module，举个例子：
```c
framework module Foundation  [extern_c] [system] {
	umbrella header "Foundation.h" // 所有要暴露的头文件
 	export *
	module * {
 		export *
 	}
 	explicit module NSDebug { //submodule
 		header "NSDebug.h"
 		export *
 	}
 }
```

swift是可以直接 `import` 一个clang module的，比如你有一些C库，需要在Swift中使用，就可以用modulemap的方式。

## Swift编译

现代化的语言几乎都抛弃了头文件，swift也不例外。问题来了，swift没有头文件又是怎么找到声明的呢？

> **编译器干了这些脏活累活** 。编译一个Swift头文件，需要解析module中所有的Swift文件，找到对应的 **声明** 。

[image:92C293F7-E207-4929-8851-A737953ACFEA-1690-000006944C115826/image_18.png]
![[image_18.png]]

当开发中难免要有Objective C和Swfit相互调用的场景，两种语言在编译的时候查找符号的方式不同，如何一起工作的呢？

**Swift引用Objective C** ：

[Swift的编译器](https://swift.org/compiler-stdlib/) 内部使用了clang，所以swift可以直接使用clang module，从而支持直接import Objective C编写的framework。

[image:315A5BDB-ACC8-4939-9B8C-563AAEA3E868-1690-000006944BED22BE/image_13.png]
![[image_13.png]]

**swift编译器会从objective c头文件里查找符号** ，头文件的来源分为两大类：

* `Bridging-Header.h` 中暴露给swfit的头文件
* framework中公开的头文件，根据编写的语言不通，可能从modulemap或者umbrella header查找
XCode提供了宏定义 `NS_SWIFT_NAME` 来让开发者定义Objective C => Swift的符号映射，可以通过Related Items -> Generate Interface来查看转换后的结果：

[image:0A6AB76B-01FB-4AB6-AA7E-D588F6CA77E2-1690-000006944BCF93B4/image_1.png]
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