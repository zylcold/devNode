#optimize #launch


## 前言

启动是 App 给用户的第一印象，启动越慢用户流失的概率就越高，良好的启动速度是用户体验不可缺少的一环。启动优化涉及到的知识点非常多面也很广，一篇文章难以包含全部，所以拆分成两部分：原理和实践。

本文从基础知识出发，先回顾一些核心概念，为后续章节做铺垫；接下来介绍 IPA 构建的基本流程，以及这个流程里可用于启动优化的点；最后大篇幅讲解 dyld3 的启动 pipeline，因为启动优化的重点还在运行时。


## IPA 构建

### pipeline
既然要构建，那么必然会有一些地方去定义如何构建，对应 Xcode 中的两个配置项：

* **Build Phase：以 Target 为维度定义了构建的流程** 。可以在 Build Phase 中插入脚本，来做一些定制化的构建，比如 CocoaPod 的拷贝资源就是通过脚本的方式完成的。
* **Build Settings：配置编译和链接相关的参数** 。特别要提到的是 other link flags 和 other c flags，因为编译和链接的参数非常多，有些需要手动在这里配置。很多项目用的 CocoaPod 做的组件化，这时候编译选项在对应的.xcconfig 文件里。

以单 Target 为例，我们来看下构建流程：

![[25a51f6e98fd46b4a45c246bd9f44fd1~tplv-k3u1fbpfcp-zoom-1.image.png]]

* 源文件(.m/.c/.swift 等)是单独编译的，输出对应的目标文件(.o)
* 目标文件和静态库/动态库一起，链接出最后的 Mach-O
* Mach-O 会被裁剪，去掉一些不必要的信息
* 资源文件如 storyboard，asset 也会编译，编译后加载速度会变快
* Mach-O 和资源文件一起，打包出最后的.app
* 对.app 签名，防篡改

### 编译

编译器可以分为两大部分：前端和后端，二者以 IR（中间代码）作为媒介。这样前后端分离，使得前后端可以独立的变化，互不影响。C 语言家族的前端是 clang，swift 的前端是 swiftc，二者的后端都是 llvm。

* 前端负责预处理，词法语法分析，生成 IR
* 后端基于 IR 做优化，生成机器码
[image:78DF94E2-D074-4C35-A5B2-465CB8DD9B3C-1452-0000725286F3A221/1fe0e1e1dc374be8bfc41178bd43636a~tplv-k3u1fbpfcp-zoom-1.image.png]
![[1fe0e1e1dc374be8bfc41178bd43636a~tplv-k3u1fbpfcp-zoom-1.image.png]]


那么如何利用编译优化启动速度呢？

**代码数量会影响启动速度，为了提升启动速度，我们可以把一些无用代码下掉** 。那怎么统计哪些代码没有用到呢？可以利用 LLVM 插桩来实现。

LLVM 的代码优化流程是一个一个 Pass，由于 LLVM 是开源的，我们可以添加一个自定义的 Pass，在函数的头部插入一些代码，这些代码会记录这个函数被调用了，然后把统计到的数据上传分析，就可以知道哪些代码是用不到的了 。

Facebook 给 LLVM 提的 [order_file](http://lists.llvm.org/pipermail/llvm-dev/2019-January/129268.html) 的 feature 就是实现了类似的插桩。

### 链接

经过编译后，我们有很多个目标文件，接着这些目标文件会和静态库，动态库一起，链接出一个 Mach-O。链接的过程并不产生新的代码，只会做一些移动和补丁。

![[bcd3a97bbc3f4ae6a681c56d328cbbaa~tplv-k3u1fbpfcp-zoom-1.image.png]]


* tbd 的全称是 text-based stub library，是因为链接的过程中只需要符号就可以了，所以 Xcode 6 开始，像 UIKit 等系统库就不提供完整的 Mach-O，而是提供一个只包含符号等信息的 tbd 文件。

举一个基于链接优化启动速度的例子：

最开始讲解 Page In 的时候，我们提到 TEXT 段的页解密很耗时，有没有办法优化呢？

**可以通过 ld 的-rename_section，把 TEXT 段中的内容，比如字符串移动到其他的段(启动路径上难免会读很多字符串)，从而规避这个解密的耗时** 。

![[146bb810c3ca4a0aa90154e1dcac1404~tplv-k3u1fbpfcp-zoom-1.image.png]]

抖音的重命名方案：

```
"-Wl,-rename_section,__TEXT,__cstring,__RODATA,__cstring",
"-Wl,-rename_section,__TEXT,__const,__RODATA,__const", 
"-Wl,-rename_section,__TEXT,__gcc_except_tab,__RODATA,__gcc_except_tab", 
"-Wl,-rename_section,__TEXT,__objc_methname,__RODATA,__objc_methname", 
"-Wl,-rename_section,__TEXT,__objc_classname,__RODATA,__objc_classname",
"-Wl,-rename_section,__TEXT,__objc_methtype,__RODATA,__objc_methtype"

复制代码
```

### 裁剪

编译完 Mach-O 之后会进行裁剪(strip)，是因为里面有些信息，如调试符号，是不需要带到线上去的。裁剪有多种级别，一般的配置如下：

* All Symbols，主二进制
* Non-Global Symbols，动态库
* Debugging Symbols，二方静态库

**为什么二方库在出静态库的时候要选择 Debugging Symbols 呢？是因为像 order_file 等链接期间的优化是基于符号的，如果把符号裁剪掉，那么这些优化也就不会生效了** 。

### 签名 & 上传

裁剪完二进制后，会和编译好的资源文件一起打包成.app 文件，接着对这个文件进行签名。签名的作用是保证文件内容不多不少，没有被篡改过。接着会把包上传到 iTunes Connect，上传后会对 `__TEXT` 段加密，加密会减弱 IPA 的压缩效果，增加包大小，也会降低启动速度 **（iOS 13 优化了加密过程，不会对包大小和启动耗时有影响）** 。


## 总结

本文回顾了 Mach-O，虚拟内存，mmap，Page In，Runloop 等基础概念，接下来介绍了 IPA 的构建流程，以及两个典型的利用编译器来优化启动的方案，最后详细的讲解了 dyld3 的启动 pipeline。

之所以花这么大篇幅讲原理，是因为任何优化都一样，只有深入理解系统运作的原理，才能找到性能的瓶颈，下一篇我们会介绍下如何利用这些原理解决实际问题。

[抖音品质建设 - iOS启动优化《原理篇》](https://juejin.cn/post/6887741815529832456)

## 前言

启动是 App 给用户的第一印象，启动越慢，用户流失的概率就越高，良好的启动速度是用户体验不可缺少的一环。启动优化涉及到的知识点非常多，面也很广，一篇文章难以包含全部，所以拆分成两部分：原理和实战，本文是实战篇。

原理篇： [抖音品质建设-iOS 启动优化《原理篇》](https://mp.weixin.qq.com/s/3-Sbqe9gxdV6eI1f435BDg)

# 如何做启动优化？

文章的正式内容开始之前，大家可以思考下，如果自己去做启动优化的，会如何去开展？

这其实是一个比较大的问题，遇到类似情况，我们都可以去把大问题拆解成几个小的问题：

* 线上用户究竟启动状况如何？
* 如何去找到可以优化的点？
* 做完优化之后，如何保持？
* 有没有一些成熟的经验可以借鉴，业界都是怎么做的？



[[启动监控时机]]
[[启动监控手段]]

# [[启动优化-最佳实践]]



[抖音品质建设 - iOS启动优化《实战篇》](https://juejin.cn/post/6921508850684133390)