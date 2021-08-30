#optimize #launch


## 前言

启动是 App 给用户的第一印象，启动越慢用户流失的概率就越高，良好的启动速度是用户体验不可缺少的一环。启动优化涉及到的知识点非常多面也很广，一篇文章难以包含全部，所以拆分成两部分：原理和实践。

本文从基础知识出发，先回顾一些核心概念，为后续章节做铺垫；接下来介绍 IPA 构建的基本流程，以及这个流程里可用于启动优化的点；最后大篇幅讲解 dyld3 的启动 pipeline，因为启动优化的重点还在运行时。

## 基本概念

### 启动的定义

启动有两种定义：

* 广义：点击图标到首页数据加载完毕
* 狭义：点击图标到 Launch Image 完全消失第一帧

不同产品的业务形态不一样，对于抖音来说，首页的数据加载完成就是视频的第一帧播放；对其他首页是静态的 App 来说，Launch Image 消失就是首页数据加载完成。由于标准很难对齐，所以我们一般使用狭义的启动定义： **即启动终点为启动图完全消失的第一帧** 。

以抖音为例，用户感受到的启动时间：
[image:B7E5E512-27FE-4923-B0F6-77E60B81A9AB-1452-0000725287409412/a5a25907e766452c8f5daf99b51bb802~tplv-k3u1fbpfcp-zoom-1.image.png]
![[a5a25907e766452c8f5daf99b51bb802~tplv-k3u1fbpfcp-zoom-1.image.png]]

> Tips：启动最佳时间是 400ms 以内，因为启动动画时长是 400ms。

这是从用户感知维度定义启动，那么代码上如何定义启动呢？Apple 在 MetricKit 中给出了官方计算方式：

* 起点：进程创建的时间
* 终点：第一个 `CA::Transaction::commit()`

> Tips： `CATransaction` 是 Core Animation 提供的一种事务机制，把一组 UI 上的修改打包，一起发给 Render Server 渲染。

### 启动的种类

根据场景的不同，启动可以分为三种：冷启动，热启动和回前台。

* 冷启动：系统里没有任何进程的缓存信息，典型的是重启手机后直接启动 App
* 热启动：如果把 App 进程杀了，然后立刻重新启动，这次启动就是热启动，因为进程缓存还在
* 回前台：大多数时候不会被定义为启动，因为此时 App 仍然活着，只不过处于 suspended 状态

那么，线上用户的冷启动多还是热启动多呢？

答案是和产品形态有关系，打开频次越高，热启动比例就越高。

### Mach-O

Mach-O 是 iOS 可执行文件的格式，典型的 Mach-O 是主二进制和动态库。Mach-O 可以分为三部分：

* Header
* Load Commands
* Data

[image:5000AFB8-6D9C-444D-B09C-50A6D8AEFE8F-1452-0000725287518BEB/23f3649b8ba345d68e22b5a3902cfa76~tplv-k3u1fbpfcp-zoom-1.image.png]
![[23f3649b8ba345d68e22b5a3902cfa76~tplv-k3u1fbpfcp-zoom-1.image.png]]

Header 的最开始是 Magic Number，表示这是一个 Mach-O 文件，除此之外还包含一些 Flags，这些 flags 会影响 Mach-O 的解析。

**Load Commands 存储 Mach-O 的布局信息** ，比如 Segment command 和 Data 中的 Segment/Section 是一一对应的。除了布局信息之外，还包含了依赖的动态库等启动 App 需要的信息。

**Data 部分包含了实际的代码和数据** ，Data 被分割成很多个 Segment，每个 Segment 又被划分成很多个 Section，分别存放不同类型的数据。

标准的三个 Segment 是 TEXT，DATA，LINKEDIT，也支持自定义：

* **TEXT** ，代码段，只读可执行，存储函数的二进制代码(__text)，常量字符串(__cstring)，Objective C 的类/方法名等信息
* **DATA** ，数据段，读写，存储 Objective C 的字符串(__cfstring)，以及运行时的元数据：class/protocol/method…
* **LINKEDIT** ，启动 App 需要的信息，如 bind & rebase 的地址，代码签名，符号表…

### dyld

dyld 是启动的辅助程序，是 in-process 的，即启动的时候会把 dyld 加载到进程的地址空间里，然后把后续的启动过程交给 dyld。dyld 主要有两个版本：dyld2 和 dyld3。

dyld2 是从 iOS 3.1 引入，一直持续到 iOS 12。dyld2 有个比较大的优化是 [dyld shared cache](http://iphonedevwiki.net/index.php/Dyld_shared_cache) ，什么是 shared cache 呢？

* shared cache 就是把系统库(UIKit 等)合成一个大的文件，提高加载性能的缓存文件。

iOS 13 开始 Apple 对三方 App 启用了 dyld3，dyld3 的最重要的特性就是启动闭包，闭包里包含了启动所需要的缓存信息，从而提高启动速度。

### 虚拟内存

内存可以分为虚拟内存和物理内存，其中物理内存是实际占用的内存，虚拟内存是在物理内存之上建立的一层逻辑地址，保证内存访问安全的同时为应用提供了连续的地址空间。

物理内存和虚拟内存以页为单位映射，但这个映射关系不是一一对应的：一页物理内存可能对应多页虚拟内存；一页虚拟内存也可能不占用物理内存。
[image:63A78996-B2C9-4E23-94EB-B8B466948620-1452-0000725286BB752A/6701d8eb76dc41eebb2ac1d5bf20b80a~tplv-k3u1fbpfcp-zoom-1.image.png]
![[6701d8eb76dc41eebb2ac1d5bf20b80a~tplv-k3u1fbpfcp-zoom-1.image.png]]


**iPhone 6s 开始，物理内存的 Page 大小是 16K，6 和之前的设备都是 4K，这是 iPhone 6 相比 6s 启动速度断崖式下降的原因之一** 。

### mmap

Mmap 的全称是 memory map，是一种内存映射技术，可以把文件映射到虚拟内存的地址空间里，这样就可以像直接操作内存那样来读写文件。 **当读取虚拟内存，其对应的文件内容在物理内存中不存在的时候，会触发一个事件：File Backed Page In，把对应的文件内容读入物理内存** 。

启动的时候，Mach-O 就是通过 mmap 映射到虚拟内存里的(如下图)。下图中部分页被标记为 zero fill，是因为全局变量的初始值往往都是 0，那么这些 0 就没必要存储在二进制里，增加文件大小。操作系统会识别出这些页，在 Page In 之后对其置为 0，这个行为叫做 zero fill。

[image:C2549A6B-FCCF-483C-B3DF-1422DF9DC83B-1452-00007252872FE78D/2e63e671214e4c9e9fbe79e142afcb00~tplv-k3u1fbpfcp-zoom-1.image.png]
![[2e63e671214e4c9e9fbe79e142afcb00~tplv-k3u1fbpfcp-zoom-1.image.png]]

### Page In

启动的路径上会触发很多次 Page In，其实也比较容易理解，因为启动的会读写二进制中的很多内容。 **Page In 会占去启动耗时的很大一部分** ，我们来看看单个 Page In 的过程：

[image:F0F94009-CF5D-4E93-8392-15B4C7C9B1C9-1452-00007252870C45DA/248ab066896543f7be1c9c9601833526~tplv-k3u1fbpfcp-zoom-1.image.png]
![[248ab066896543f7be1c9c9601833526~tplv-k3u1fbpfcp-zoom-1.image.png]]

* MMU 找到空闲的物理内存页面
* 触发磁盘 IO，把数据读入物理内存
* 如果是 TEXT 段的页，要进行解密
* 对解密后的页，进行签名验证

其中 **解密是大头，IO 其次。**

为什么要解密呢？因为 iTunes Connect 会对上传 Mach-O 的 TEXT 段进行加密，防止 IPA 下载下来就直接可以看到代码。这也就是为什么逆向里会有个概念叫做“砸壳”，砸的就是这一层 TEXT 段加密。 **iOS 13 对这个过程进行了优化，Page In 的时候不需要解密了** 。

### 二进制重排

既然 Page In 耗时，有没有什么办法优化呢？启动具有 **局部性特征** ，即只有少部分函数在启动的时候用到，这些函数在二进制中的分布是零散的，所以 Page In 读入的数据利用率并不高。如果我们可以把启动用到的函数排列到二进制的连续区间，那么就可以减少 Page In 的次数，从而优化启动时间：

以下图为例，方法 1 和方法 3 是启动的时候用到的，为了执行对应的代码，就需要两次 Page In。假如我们把方法 1 和 3 排列到一起，那么只需要一次 Page In，从而提升启动速度。

[image:D620B97E-4A23-4501-A35F-EBCDD76FEF12-1452-0000725286CDE0DB/ead668453cd34520bcd889d16fca12d0~tplv-k3u1fbpfcp-zoom-1.image.png]
![[ead668453cd34520bcd889d16fca12d0~tplv-k3u1fbpfcp-zoom-1.image.png]]

链接器 ld 有个参数-order_file 支持按照符号的方式排列二进制。获取启动时候用到的符号的有很多种方式，感兴趣的同学可以看看抖音之前的文章： [基于二进制文件重排的解决方案 APP 启动速度提升超 15%](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&amp;mid=2247485101&amp;idx=1&amp;sn=abbbb6da1aba37a04047fc210363bcc9&amp;scene=21&amp;token=2051547505&amp;lang=zh_CN) 。

## IPA 构建

### pipeline

既然要构建，那么必然会有一些地方去定义如何构建，对应 Xcode 中的两个配置项：

* **Build Phase：以 Target 为维度定义了构建的流程** 。可以在 Build Phase 中插入脚本，来做一些定制化的构建，比如 CocoaPod 的拷贝资源就是通过脚本的方式完成的。
* **Build Settings：配置编译和链接相关的参数** 。特别要提到的是 other link flags 和 other c flags，因为编译和链接的参数非常多，有些需要手动在这里配置。很多项目用的 CocoaPod 做的组件化，这时候编译选项在对应的.xcconfig 文件里。

以单 Target 为例，我们来看下构建流程：

[image:2AE9B9E9-6BB1-46FA-BDAE-5ABFF1DEEACF-1452-00007252871FBD78/25a51f6e98fd46b4a45c246bd9f44fd1~tplv-k3u1fbpfcp-zoom-1.image.png]
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

[image:27BDC93A-04A6-4FA1-AC05-BDFA2E5AA675-1452-0000725286A49015/bcd3a97bbc3f4ae6a681c56d328cbbaa~tplv-k3u1fbpfcp-zoom-1.image.png]
![[bcd3a97bbc3f4ae6a681c56d328cbbaa~tplv-k3u1fbpfcp-zoom-1.image.png]]


* tbd 的全称是 text-based stub library，是因为链接的过程中只需要符号就可以了，所以 Xcode 6 开始，像 UIKit 等系统库就不提供完整的 Mach-O，而是提供一个只包含符号等信息的 tbd 文件。

举一个基于链接优化启动速度的例子：

最开始讲解 Page In 的时候，我们提到 TEXT 段的页解密很耗时，有没有办法优化呢？

**可以通过 ld 的-rename_section，把 TEXT 段中的内容，比如字符串移动到其他的段(启动路径上难免会读很多字符串)，从而规避这个解密的耗时** 。
[image:228B59D3-CC74-42B2-BE56-4436E87E2EB0-1452-0000725286E0497E/146bb810c3ca4a0aa90154e1dcac1404~tplv-k3u1fbpfcp-zoom-1.image.png]
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

## dyld3 启动流程

Apple 在 iOS 13 上对第三方 App 启用了 dyld3， [官方数据](https://developer.apple.com/support/app-store/) 显示，过去四年新发布的设备中有 93%的设备是 iOS 13，所以我们重点看下 dyld3 的启动流程。

### Before dyld

用户点击图标之后，会发送一个系统调用 execve 到内核，内核创建进程。 **接着会把主二进制 mmap 进来，读取 load command 中的 LC_LOAD_DYLINKER，找到 dyld 的的路径** 。然后 mmap dyld 到虚拟内存，找到 dyld 的入口函数 `_dyld_start` ，把 PC 寄存器设置成 `_dyld_start` ，接下来启动流程交给了 dyld。

注意这个过程都是在内核态完成的，这里提到了 PC 寄存器，PC 寄存器存储了下一条指令的地址，程序的执行就是不断修改和读取 PC 寄存器来完成的。

### dyld

#### 创建启动闭包

dyld 会首先创建启动闭包，闭包是一个缓存，用来提升启动速度的。既然是缓存，那么必然不是每次启动都创建的，只有在重启手机或者更新/下载 App 的第一次启动才会创建。 **闭包存储在沙盒的 tmp/com.apple.dyld 目录，清理缓存的时候切记不要清理这个目录** 。

闭包是怎么提升启动速度的呢？我们先来看一下闭包里都有什么内容：

* dependends，依赖动态库列表
* fixup：bind & rebase 的地址
* initializer-order：初始化调用顺序
* optimizeObjc: Objective C 的元数据
* 其他：main entry, uuid…

动态库的依赖是树状的结构，初始化的调用顺序是先调用树的叶子结点，然后一层层向上，最先调用的是 libSystem，因为他是所有依赖的源头。
[image:87350DAC-5780-48AB-AE1A-7EBEABF58DA8-1452-0000725287625B08/c98498fff55540edaf8c01c522183bc0~tplv-k3u1fbpfcp-zoom-1.image.png]
![[c98498fff55540edaf8c01c522183bc0~tplv-k3u1fbpfcp-zoom-1.image.png]]

为什么闭包能提高启动速度呢？

**因为这些信息是每次启动都需要的，把信息存储到一个缓存文件就能避免每次都解析，尤其是 Objective C 的运行时数据（Class/Method...）解析非常慢** 。

#### fixup

有了闭包之后，就可以用闭包启动 App 了。这时候很多动态库还没有加载进来，会首先对这些动态库 mmap 加载到虚拟内存里。接着会对每个 Mach-O 做 fixup，包括 Rebase 和 Bind。

* **Rebase：修复内部指针** 。这是因为 Mach-O 在 mmap 到虚拟内存的时候，起始地址会有一个随机的偏移量 slide，需要把内部的指针指向加上这个 slide。
* **Bind：修复外部指针** 。这个比较好理解，因为像 printf 等外部函数，只有运行时才知道它的地址是什么，bind 就是把指针指向这个地址。

举个例子：一个 Objective C 字符串@"1234"，编译到最后的二进制的时候是会存储在两个 section 里的

* `__TEXT，__cstring` ，存储实际的字符串"1234"
* `__DATA，__cfstring` ，存储 Objective C 字符串的元数据，每个元数据占用 32Byte，里面有两个指针：内部指针，指向 `__TEXT，__cstring` 中字符串的位置；外部指针 isa，指向类对象的，这就是为什么可以对 Objective C 的字符串字面量发消息的原因。

如下图，编译的时候，字符串 1234 在 `__cstring` 的 0x10 处，所以 DATA 段的指针指向 0x10。但是 mmap 之后有一个偏移量 slide=0x1000，这时候字符串在运行时的地址就是 0x1010，那么 DATA 段的指针指向就不对了。 **Rebase 的过程就是把指针从 0x10，加上 slide 变成 0x1010。运行时类对象的地址已经知道了，bind 就是把 isa 指向实际的内存地址** 。

#### LibSystem Initializer

Bind & Rebase 之后，首先会执行 LibSystem 的 Initializer，做一些最基本的初始化：

* 初始化 libdispatch
* 初始化 objc runtime，注册 sel，加载 category

注意这里没有初始化 objc 的类方法等信息，是因为启动闭包的缓存数据已经包含了 optimizeObjc。

#### Load & Static Initializer

接下来会进行 main 函数之前的一些初始化，主要包括+load 和 static initializer。这两类初始化函数都有个特点： **调用顺序不确定，和对应文件的链接顺序有关系** 。那么就会存在一个隐藏的坑：有些注册逻辑在+load 里，对应会有一些地方读取这些注册的数据，如果在+load 中读取，很有可能读取的时候还没有注册。

那么，如何找到代码里有哪些 load 和 static initializer 呢？

在 Build Settings 里可以配置 write linkmap，这样在生成的 linkmap 文件里就可以找到有哪些文件里包含 load 或者 static initializer：

* `__mod_init_func` ，static initializer
* `__objc_nlclslist` ，实现+load 的类
* `__objc_nlcatlist` ，实现+load 的 Category

#### load 举例

如果+load 方法里的内容很简单，会影响启动时间么？比如这样的一个+load 方法？

```
+ (void)load 
{
    printf("1234");
}
复制代码
```

编译完了之后，这个函数会在二进制中的 TEXT 两个段存在： `__text` **存函数二进制，** `cstring` **存储字符串 1234** 。为了执行函数，首先要访问 `__text` 触发一次 Page In 读入物理内存，为了打印字符串，要访问 `__cstring` ，还会触发一次 Page In。

* 为了执行这个简单的函数， **系统要额外付出两次 Page In 的代价** ，所以 load 函数多了，page in 会成为启动性能的瓶颈。

#### static initializer 产生的条件

静态初始化是从哪来的呢？以下几种代码会导致静态初始化

* `__attribute__((constructor))`
* `static class object`
* `static object in global namespace`

注意，并不是所有的 static 变量都会产生静态初始化，编译器很智能，对于在编译期间就能确定的变量是会直接 inline。

```
//会产生静态初始化
class Demo{ 
static const std::string var_1; 
};
const std::string var_2 = "1234"; 
static Logger logger;
//不会产生静态初始化
static const int var_3 = 4; 
static const char * var_4 = "1234";
复制代码
```

std::string 会合成 static initializer 是因为初始化的时候必须执行构造函数，这时候编译器就不知道怎么做了，只能延迟到运行时～

### UIKit Init

+load 和 static initializer 执行完毕之后，dyld 会把启动流程交给 App，开始执行 main 函数。main 函数里要做的最重要的事情就是初始化 UIKit。UIKit 主要会做两个大的初始化：

* 初始化 UIApplication
* 启动主线程的 Runloop

**由于主线程的 dispatch_async 是基于 runloop 的，所以在+load 里如果调用了 dispatch_async 会在这个阶段执行。**

#### Runloop

线程在执行完代码就会退出，很明显主线程是不能退出的，那么就需要一种机制：事件来的时候执行任务，否则让线程休眠，Runloop 就是实现这个功能的。

Runloop 本质上是一个 `While` 循环，在图中橙色部分的 `mach_msg_trap` 就是触发一个系统调用，让线程休眠，等待事件到来，唤醒 Runloop，继续执行这个 `while` 循环。

Runloop 主要处理几种任务：Source0，Source1，Timer，GCD MainQueue，Block。在循环的合适时机，会以 Observer 的方式通知外部执行到了哪里。

那么，Runloop 与启动又有什么关系呢？

* **App 的 LifeCycle 方法是基于 Runloop 的 Source0 的**
* **首帧渲染是基于 Runloop Block 的**

Runloop 在启动上主要有几点应用：

* 精准统计启动时间
* 找到一个时机，在启动结束去执行一些预热任务
* 利用 Runloop 打散耗时的启动预热任务

> **Tips:** 会有一些逻辑要在启动之后 delay 一小段时间再回到主线程上执行，对于性能较差的设备，主线程 Runloop 可能一直处于忙的状态，所以这个 delay 的任务并不一定能按时执行。

### AppLifeCycle

UIKit 初始化之后，就进入了我们熟悉的 UIApplicationDelegate 回调了，在这些会调里去做一些业务上的初始化：

* `willFinishLaunch`

* `didFinishLaunch`

* `didFinishLaunchNotification`

要特别提一下 `didFinishLaunchNotification` ，是因为大家在埋点的时候通常会忽略还有这个通知的存在，导致把这部分时间算到 UI 渲染里。

### First Frame Render

一般会用 Root Controller 的 viewDidApper 作为渲染的终点，但其实这时候首帧已经渲染完成一小段时间了，Apple 在 MetricsKit 里对启动终点定义是第一个 `CA::Transaction::commit()` 。

什么是 CATransaction 呢？我们先来看一下渲染的大致流程

iOS 的渲染是在一个单独的进程 RenderServer 做的，App 会把 Render Tree 编码打包给 RenderServer，RenderServer 再调用渲染框架(Metal/OpenGL ES)来生成 bitmap，放到帧缓冲区里，硬件根据时钟信号读取帧缓冲区内容，完成屏幕刷新。CATransaction 就是把一组 UI 上的修改，合并成一个事务，通过 commit 提交。

渲染可以分为四个步骤

* **Layout** （布局），源头是 Root Layer 调用 `[CALayer layoutSubLayers]` ，这时候 `UIViewController` 的 `viewDidLoad` 和 `LayoutSubViews` 会调用， `autolayout` 也是在这一步生效
* **Display** （绘制），源头是 Root Layer 调用 `[CALayer display]` ，如果 View 实现了 `drawRect` 方法，会在这个阶段调用
* **Prepare** （准备），这个过程中会完成图片的解码
* **Commit** （提交），打包 Render Tree 通过 XPC 的方式发给 Render Server

### 启动 Pipeline

详细回顾下整个启动过程，以及各个阶段耗时的影响因素：

1. 点击图标，创建进程
2. mmap 主二进制，找到 dyld 的路径
3. mmap dyld，把入口地址设为 `_dyld_start`
4. 重启手机/更新/下载 App 的第一次启动，会创建启动闭包
5. 把没有加载的动态库 mmap 进来，动态库的数量会影响这个阶段
6. 对每个二进制做 bind 和 rebase，主要耗时在 Page In，影响 Page In 数量的是 objc 的元数据
7. 初始化 objc 的 runtime，由于闭包已经初始化了大部分，这里只会注册 sel 和装载 category
8. +load 和静态初始化被调用，除了方法本身耗时，这里还会引起大量 Page In
9. 初始化 `UIApplication` ，启动 Main Runloop
10. 执行 `will/didFinishLaunch` ，这里主要是业务代码耗时
11. Layout， `viewDidLoad` 和 `Layoutsubviews` 会在这里调用， `Autolayout` 太多会影响这部分时间
12. Display， `drawRect` 会调用
13. Prepare，图片解码发生在这一步
14. Commit，首帧渲染数据打包发给 RenderServer，启动结束

## dyld2

dyld2 和 dyld3 的主要区别就是没有启动闭包，就导致每次启动都要：

* 解析动态库的依赖关系
* 解析 LINKEDIT，找到 bind & rebase 的指针地址，找到 bind 符号的地址
* 注册 objc 的 Class/Method 等元数据，对大型工程来说，这部分耗时会很长

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

对应着本文的三大模块： **监控** ， **工具** 和 **最佳实践。**

# 监控

## 启动埋点

既然要监控，那么就要能够在代码里获取到启动时长。启动的起点大家采用的方案都一样：进程创建的时间。

启动的终点对应用户感知到的 Launch Image 消失的第一帧，抖音采用的方案如下：
[image:CC51D733-F560-4ED1-8B1B-2273E410978A-1452-0000725285608F23/907d8cde9d2745549739cb03c979fb76~tplv-k3u1fbpfcp-zoom-1.image.png]
![[907d8cde9d2745549739cb03c979fb76~tplv-k3u1fbpfcp-zoom-1.image.png]]

* iOS 12 及以下：root viewController 的 viewDidAppear
* iOS 13+：applicationDidBecomeActive

Apple 官方的统计方式是第一个 `CA::Transaction::commit` ，但对应的实现在系统框架内部，抖音的方式已经非常接近这个点了。

## 分阶段

只有一个启动耗时的埋点在排查线上问题的时候显然是不够的，可以通过 **分阶段和单点埋** 点结合，下面是这是目前抖音的监控方案：

+load、initializer 的调用顺序和链接顺序有关，链接顺序默认按照 CocoaPod 的 Pod 命名升序排列，所以取一个命名为 AAA 开头既可以让某个 +load、initializer 第一个被执行。

## 无侵入监控

公司的 APM 团队提供了一种无侵入的启动监控方案，方案将启动流程拆分成几个粒度比较粗的与业务无关的阶段：进程创建，最早的 +load，didFinishLuanching 开始和首屏首次绘制完成。
[image:F86113EF-D0B9-4D44-9D14-1194AE6F226D-1452-0000725285EA39F3/a920e115e1ae4e918117c883565becde~tplv-k3u1fbpfcp-watermark.image.png]
![[a920e115e1ae4e918117c883565becde~tplv-k3u1fbpfcp-watermark.image.png]]

前三个时间点无侵入获取较为简单

* 进程创建：通过 `sysctl` 系统调用拿到进程创建的时间戳
* 最早的 +load：和上面的分阶段监控一样，通过 AAA 为前缀命名 Pod，让 +load 第一个被执行
* didFinishLaunching：监控 SDK 初始化一般在启动的很早期，用监控 SDK 的初始化时间作为 didFinishLaunching 的时间

首屏渲染完成时间我们希望和 `MetricKit` 对齐，即获取到 `CA::Transaction::commit()` 方法被调用的时间。

通过 Runloop 源码分析和线下调试，我们发现 `CA::Transaction::commit()` ， `CFRunLoopPerformBlock` ， `kCFRunLoopBeforeTimers` 这三个时机的顺序从早到晚依次是：
[image:731D25D7-D240-47ED-ADA8-23AB27F2F5F7-1452-000072528636F5B3/eed6c7cc2c85460892071a44ea4b0a3e~tplv-k3u1fbpfcp-zoom-1.image.png]
![[eed6c7cc2c85460892071a44ea4b0a3e~tplv-k3u1fbpfcp-zoom-1.image.png]]

可以通过在 didFinishLaunch 中向 Runloop 注册 block 或者 BeforeTimer 的 Observer 来获取上图中两个时间点的回调，代码如下：

```
//注册block
CFRunLoopRef mainRunloop = [[NSRunLoop mainRunLoop] getCFRunLoop];
CFRunLoopPerformBlock(mainRunloop,NSDefaultRunLoopMode,^(){
    NSTimeInterval stamp = [[NSDate date] timeIntervalSince1970];
    NSLog(@"runloop block launch end:%f",stamp);
});
//注册kCFRunLoopBeforeTimers回调
CFRunLoopRef mainRunloop = [[NSRunLoop mainRunLoop] getCFRunLoop];
CFRunLoopActivity activities = kCFRunLoopAllActivities;
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, activities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
    if (activity == kCFRunLoopBeforeTimers) {
        NSTimeInterval stamp = [[NSDate date] timeIntervalSince1970];
        NSLog(@"runloop beforetimers launch end:%f",stamp);
        CFRunLoopRemoveObserver(mainRunloop, observer, kCFRunLoopCommonModes);
    }
});
CFRunLoopAddObserver(mainRunloop, observer, kCFRunLoopCommonModes);
复制代码
```

经过实测，我们最后选择的无侵入获取首屏渲染方案是：

1. iOS13（含）以上的系统采用 `runloop` 中注册一个 `kCFRunLoopBeforeTimers` 的回调获取到的 App 首屏渲染完成的时机更准确。
2. iOS13 以下的系统采用 `CFRunLoopPerformBlock` 方法注入 block 获取到的 App 首屏渲染完成的时机更准确。

## 监控周期
[image:8D8E8174-0942-47FC-899A-D812E1F6F470-1452-00007252860F733B/0e53681bd7f1460194039623e402b839~tplv-k3u1fbpfcp-zoom-1.image.png]
![[0e53681bd7f1460194039623e402b839~tplv-k3u1fbpfcp-zoom-1.image.png]]

App 的生命周期可以分为三个阶段：研发，灰度和线上，不同阶段监控的目的和方式都不一样。

### 研发阶段

研发阶段的监控主要目的是防止劣化，对应着会有线下的自动化监控，通过实际的启动性能测试来尽早地发现和解决问题，抖音的线下自动化监控流程图如下：
[image:BDB7B086-6616-4E98-9DD0-FF45D4867489-1452-0000725285D18722/a03f32a95d8c4fd0a2171f7a8618146c~tplv-k3u1fbpfcp-zoom-1.image.png]
![[a03f32a95d8c4fd0a2171f7a8618146c~tplv-k3u1fbpfcp-zoom-1.image.png]]

由定时任务触发，先 release 模式下打包，接着跑一次自动化测试，测试完毕后会上报测试结果，方便通过看板来跟踪整体的变化趋势。

如果发现有劣化，会先发一条报警信息，接着通过二分查找的方式找到对应的劣化 MR，然后自动跑火焰图和 Instrument 来辅助定位问题。

那么如何保证测试的结果是稳定可靠的呢？

答案就是控制变量：

* 关闭 iCloud & 不登录 AppleID & 飞行模式
* 风扇降温，且用 MFI 认证数据线
* 重启手机和开始下一次测试之前静置一段时间
* 多次测量取平均值 & 计算方差
* Mock 启动过程中的 AB 变量

实践下来，我们发现 iPhone 8 的稳定性最好，其次是 iPhone X，iPhone 6 的稳定性很差。

除了自动化测试，在研发流程上还可以加一些准入，来防止启动劣化，这些准入包括

* **新增动态库**
* **新增 +load 和静态初始化**
* **新增启动任务 Code Review**

不建议做细粒度的 Code Review，除非对相关业务很了解，否则一般肉眼看不出会不会有劣化。

### 线上 & 灰度

灰度和线上的策略是相似的， **主要看的是大盘数据和配置报警** ，大盘监控和报警和公司的基建有很大关系，如果没有对应基建 Xcode MetricKit 本身也可以看到启动耗时：打开 Xcode -> Window -> Origanizer -> Launch Time

大盘数据本身是统计学的，会有些统计学的规律：

* **发版本的前几天启动速度比较慢** ，这是因为 iOS 13 后更新 App 的第一次启动要创建启动闭包，这个过程是比较慢的
* **新版本发布会导致老版本 pct50 变慢** ，因为性能差的设备升级速度慢，导致老版本性能差设备比例变高
* **采样率调整会影响 pct50** ，比如某些地区的 iPhone 6 比例较高，如果这些地区采样率提高会导致大盘性能差的设备比例提高。

基于这些背景，我们一般会通过控制变量的方式：拆地区，机型，版本，有时候甚至要根据时间看启动耗时的趋势。

# 工具

完成了监控之后，我们需要找到一些可以优化的点，就需要用到工具。主要包括两大类： **Instrument 和自研** 。

## Time Profiler

Time Profiler 是大家日常性能分析中用的比较多的工具，通常会选择一个时间段，然后聚合分析调用栈的耗时。但 **Time Profiler 其实只适合粗粒度的分析** ，为什么这么说呢？我们来看下它的实现原理：

**默认 Time Profiler 会 1ms 采样一次，只采集在运行线程的调用栈，最后以统计学的方式汇总** 。比如下图中的 5 次采样中，method3 都没有采样到，所以最后聚合到的栈里就看不到 method3。所以 Time Profiler 中的看到的时间，并不是代码实际执行的时间，而是栈在采样统计中出现的时间。
[image:986A0AE9-D81F-4FC8-BAB6-3C21FD3621FC-1452-0000725285878893/2675a2c20cda42c1bbd9e8afb9872c48~tplv-k3u1fbpfcp-zoom-1.image.png]
![[2675a2c20cda42c1bbd9e8afb9872c48~tplv-k3u1fbpfcp-zoom-1.image.png]]

Time Profiler 支持一些额外的配置， **如果统计出来的时间和实际的时间相差比较多，可以尝试开启** ：

* High Frequency，降低采样的时间间隔
* Record Kernel Callstacks，记录内核的调用栈
* Record Waiting Thread，记录被 block 的线程

## System Trace

既然 Time Profiler 支持粗粒度的分析，那么有没有什么精细化的分析工具呢？答案就是 System Trace。
[image:F7D993D5-02E3-464D-AAD7-8D55DA21596F-1452-0000725285998746/8635a869aa06443c9ff91ecafb09d024~tplv-k3u1fbpfcp-zoom-1.image.png]
![[8635a869aa06443c9ff91ecafb09d024~tplv-k3u1fbpfcp-zoom-1.image.png]]

既然要精细化分析，那么我们就需要标记出一小段时间，可以用 Point of interest 来标记。除此之外，System Trace 分析虚拟内存和线程状态都很管用：

* Virtual Memory： **主要关注 Page In** 这个事件，因为启动路径上有很多次 Page In，且相对耗时
* Thread State： **主要关注挂起和抢占两个状态，记住主线程不是一直在运行的**
* System Load 线程有优先级，高优先级的线程不应该超过系统核心数量

## os_signpost

os_signpost 是 iOS 12 推出的用于在 instruments 里标记时间段的 API，性能非常高，可以认为对启动无影响。结合最开始讲的分阶段监控，我们可以在 Instrument 把启动划分成多个阶段，和其他模板一起分析具体问题：
[image:3383ACD9-61D6-468A-AA0D-BB3D6D1E61CF-1452-0000725285AB1CC2/4a4b3c8941174428bcae89c9d05b76b0~tplv-k3u1fbpfcp-zoom-1.image.png]
![[4a4b3c8941174428bcae89c9d05b76b0~tplv-k3u1fbpfcp-zoom-1.image.png]]

结合 swizzle，os_signpost 可以发挥出意想不到的效果，比如 hook 所有的 load 方法，来分析对应耗时，又比如 hook UIImage 对应方法，来统计启动路径上用到的图片加载耗时。

## 其他 Instrument 模板

除了这些，还有几个模板是比较常用的：

* **Static Initializer** ：分析 C++ 静态初始化
* **App Launch** ：Xcode 11 后新出的模板，可以认为是 Time Profiler + System Trace
* **Custom Instrument** ：自定义 Instrument，最简单是用 os_signpost 作为模板的数据源，自己做一些简单的定制化展示，具体可参考 WWDC 的相关 Session。

## 火焰图

**火焰图用来分析时间相关的性能瓶颈非常有用** ，可以直接把业务代码的耗时绘制出来。此外，火焰图可以自动化生成然后 diff，所以可以用于自动化归因。

火焰图有两种常见实现方式

* hook objc_msgSend
* 编译期插桩

本质上都是在方法的开始和末尾打两个点，就知道这个方法的耗时，然后转换成 Chrome 的标准的 json 格式就可以分析了。注意就算用 mmap 来写文件，仍然会有一些误差，所以找到的问题并不一定是问题，需要二次确认。

# 最佳实践

## 整体思路

优化的整体思路其实就四步：
[image:167C7CAD-1CDA-49FF-A4E4-F2B37B6584A1-1452-0000725286730E8A/abe2e95368914aabbd06594917b16a4c~tplv-k3u1fbpfcp-zoom-1.image.png]
![[abe2e95368914aabbd06594917b16a4c~tplv-k3u1fbpfcp-zoom-1.image.png]]

1. **删** 掉启动项，最直接
2. 如果不能删除，尝试 **延迟** ，延迟包括第一次访问以及启动结束后找个合适的时间预热
3. 不能延迟的可以尝试 **并发** ，利用好多核多线程
4. 如果并发也不行，可以尝试让代码执行 **更快**

**这块会以 Main 函数做分界线，看下 Main 函数前后的优化方案；接着介绍如何优化 Page In；最后讲解一些非常规的优化方案，这些方案对架构的要求比较高** 。

## Main 之前

Main 函数之前的启动流程如下：

* 加载 dyld
* 创建启动闭包（更新 App/重启手机需要）
* 加载动态库
* Bind & Rebase & Runtime 初始化
* +load 和静态初始化
[image:870B19BB-AD39-4B48-8B72-A88C31090E2E-1452-000072528501ABC4/a1857c80db58462faa2eee20d72c07ce~tplv-k3u1fbpfcp-zoom-1.image.png]
![[a1857c80db58462faa2eee20d72c07ce~tplv-k3u1fbpfcp-zoom-1.image.png]]

### 动态库

减少动态库数量可以加减少启动闭包创建和加载动态库阶段的耗时，官方建议动态库数量小于 6 个。

**推荐的方式是动态库转静态库** ，因为还能额外减少包大小。另外一个方式是合并动态库，但实践下来可操作性不大。最后一点要提的是， **不要链接那些用不到的库** （包括系统），因为会拖慢创建闭包的速度。

### 下线代码

下线代码可以减少 Rebase & Bind & Runtime 初始化的耗时。那么如何找到用不到的代码，然后把这些代码下线呢？ **可以分为静态扫描和线上统计两种方式** 。

最简单的静态扫描是基于 AppCode，但是项目大了之后 AppCode 的索引速度非常慢，另外的一种静态扫描是基于 Mach-O 的：

* `_objc_selrefs` 和 `_objc_classrefs` 存储了引用到的 sel 和 class
* `__objc_classlist` 存储了所有的 sel 和 class

二者做个差集就知道那些类/sel 用不到，但 **objc 支持运行时调用，删除之前还要在二次确认** 。

还有一种统计无用代码的方式是用线上的数据统计，主流的方案有三种：

* ViewConteroller 渗透率，hook 对应的声明周期方法即可统计
* Class 渗透率，遍历运行时的所有类，通过 Objective C Runtime 的标志位判断类是否被访问
* 行级渗透率，需要用编译期插桩，对包大小和执行速度均有损。

前两种是 ROI 较高的方案，绝大多数时候 Class 级别的渗透率足够了。

### +load 迁移

**+load 除了方法本身的耗时，还会引起大量 Page In** ，另外 +load 的存在对 App 稳定性也是冲击，因为 Crash 了捕获不到。

举个例子，很多 DI 的容器需要把协议绑定到类，所以需要在启动的早期(+load)里注册：

```
+ (void)load
{
    [DICenter bindClass:IMPClass toProtocol:@protocol(SomeProcotol)]
}
```

本质上只要知道协议和类的对应关系即可，利用 clang attribute，这个过程可以迁移到编译期：

```objectivec
typedef struct{
    const char * cls;
    const char * protocol;
}_di_pair;
#if DEBUG
#define DI_SERVICE(PROTOCOL_NAME,CLASS_NAME)\
__used static Class<PROTOCOL_NAME> _DI_VALID_METHOD(void){\
    return [CLASS_NAME class];\
}\
__attribute((used, section(_DI_SEGMENT "," _DI_SECTION ))) static _di_pair _DI_UNIQUE_VAR = \
{\
_TO_STRING(CLASS_NAME),\
_TO_STRING(PROTOCOL_NAME),\
};\
#else
__attribute((used, section(_DI_SEGMENT "," _DI_SECTION ))) static _di_pair _DI_UNIQUE_VAR = \
{\
_TO_STRING(CLASS_NAME),\
_TO_STRING(PROTOCOL_NAME),\
};\
#endif
```

原理很简单： **宏提供接口，编译期把类名和协议名写到二进制的指定段里，运行时把这个关系读出来就知道协议是绑定到哪个类了** 。

有同学会注意到有个无用的方法 `_DI_VALID_METHOD` ，这个方法只在 debug 模式下存在，为了让编译器保证类型安全。

### 静态初始化迁移

静态初始化和 +load 方法一样也会引起大量 Page In，一般来自 C++代码，比如网络或者特效的库。另外 **有些静态初始化是通过头文件引入进来的** ，可以通过预处理来确认。

几个典型的迁移思路：

* std:string 转换成 const char *
* 静态变量移动到方法内部，因为方法内部的静态变量会在方法第一次调用的时候初始化

```c++
//Bad
namespace {
    static const std::string bucket[] = {"apples", "pears", "meerkats"};
}
const std::string GetBucketThing(int i) {
     return bucket[i];
}
//Good
std::string GetBucketThing(int i) {
  static const std::string bucket[] = {"apples", "pears", "meerkats"};
  return bucket[i];
}
```

## Main 之后

### 启动器

启动是需要一个框架来管控的，抖音采用了轻量级的中心式方案：

* 有个启动任务的配置仓，里面只包含启动任务的顺序和线程
* 业务仓实现协议 BootTask，表明这是个启动任务

启动任务的执行流程如下：
[image:13E1ABAF-D5E2-4C2E-862A-301DA1F7A397-1452-0000725285755C3B/637af7bc814844df98555914d7f78a03~tplv-k3u1fbpfcp-zoom-1.image.png]
![[637af7bc814844df98555914d7f78a03~tplv-k3u1fbpfcp-zoom-1.image.png]]

为什么需要启动器呢？

* **全局并发调度** ：比如 AB 任务并发，C 任务等待 AB 执行完毕，框架调度还能减少线程数量和控制优先级
* **延迟执行** ：提供一些时机，业务可以做预热性质的初始化
* **精细化监控** ：所有任务的耗时都能监控到，线下自动化监控也能受益
* **管控** ：启动任务的顺序调整，新增/删除都能通过 Code Review 管控

### 三方 SDK

有些三方 SDK 的启动耗时很高，比如 Fabric，抖音下线了 Fabric 后启动速度 pct50 快了 70ms 左右。

除了下线，很多 SDK 是可以延迟的，比如分享和登录的 SDK。此外，在接入 SDK 之前可以先评估下对启动性能的影响，如果影响较大是可以反馈给 SDK 的提供方去修改的，尤其是付费的 SDK，他们其实很愿意配合做一些修改。

### 高频次方法

有些方法的单个耗时不高，但是在启动路径上会调用很多次的，这种累计起来的耗时也不低，比如读 Info.plist 里面的配置：

```
+ (NSString *)plistChannel
{
    return [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CHANNEL_NAME"];
}
复制代码
```

修改的方式很简单，加一层内存缓存即可，这种问题在 TimeProfiler 里时间段选长一些往往就能看出来。

### 锁

锁之所以会影响启动时间，是因为有时候子线程先持有了锁， **主线程就需要等待子线程锁释放。还要警惕系统会有很多隐藏的全局锁** ，比如 dyld 和 Runtime。举个例子：

下图是 `UIImage imageNamed` 引起的主线程 block:
[image:23D2CD30-52F0-4801-B91E-B8D3D5121769-1452-0000725285FC9589/f814285cecf541e2897120af90f3947f~tplv-k3u1fbpfcp-zoom-1.image.png]
![[f814285cecf541e2897120af90f3947f~tplv-k3u1fbpfcp-zoom-1.image.png]]

通过右侧的堆栈能看到，imageNamed 触发了 dlopen，dlopen 会等待 dyld 的全局锁。 **通过 System Trace 的 Thread State 
Event，可以找到线程被 blocked 的下一个事件** ，这个事件表明了线程重新可以运行，原因就是其他线程释放了锁：
[image:499F606D-EB4C-4139-AAC3-009AEAD8231D-1452-0000725286221513/8e9ee3a2cd6a42fc94a6e05e10cbeab0~tplv-k3u1fbpfcp-zoom-1.image.png]
![[8e9ee3a2cd6a42fc94a6e05e10cbeab0~tplv-k3u1fbpfcp-zoom-1.image.png]]

接下来通过分析后台线程这个时间在做什么，就知道为什么会持有锁，如何优化了。

### 线程数量

线程的数量和优先级都会影响启动时间。可以通过设置 QoS 来配置优先级，两个高优的 QoS 是 User Interactive/Initiated，启动的时候， **需要主线程等待的子线程任务都应该设置成高优的** 。

**高优的线程数量不应该多于 CPU 核心数量** ，可以通过 System Trace 的 System Load 来分析这种情况。

```
/GCD
dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, -1);
dispatch_queue_t queue = dispatch_queue_create("com.custom.utility.queue", attr);
//NSOperationQueue
operationQueue.qualityOfService = NSQualityOfServiceUtility
复制代码
```

线程的数量也会影响启动时间，但 iOS 中是不太好全局管控线程的，比如二/三方库要起后台线程就不太好管控，不过业务上的线程可以通过启动任务管控。

> 线程多没关系，只要同时 **并发执行的不多就好** ，大家可以利用 System Trace 来看看上下文切换耗时，确认线程数量是否是启动的瓶颈。

### 图片

启动难免会用到很多图，有没有办法优化图片加载的耗时呢？

**用 Asset 管理图片而不是直接放在 bundle 里** 。Asset 会在编译期做优化，让加载的时候更快，此外在 Asset 中加载图片是要比 Bundle 快的，因为 UIImage imageNamed 要遍历 Bundle 才能找到图。 **加载 Asset 中图的耗时主要在在第一次张图，因为要建立索引** ，可以通过把启动的图放到一个小的 Asset 里来减少这部分耗时。

每次创建 UIImage 都需要 IO，在首帧渲染的时候会解码。所以可以通过提前子线程预加载（创建 UIImage）来优化这部分耗时。

如下图，启动只有到了比较晚的阶段“RootWindow 创建”和“首帧渲染”才会用到图片， **所以可以在启动的早期开预加载的子线程启动任务** 。
[image:2C65DD18-C00D-4249-AFF7-5FB770D52F7C-1452-00007252864989D6/f19a27a060f94511a345954724efba8d~tplv-k3u1fbpfcp-zoom-1.image.png]
![[f19a27a060f94511a345954724efba8d~tplv-k3u1fbpfcp-zoom-1.image.png]]

### Fishhook

Fishhook 是一个用来 hook C 函数的库，但这个库的第一次调用耗时很高，最好 **不要带到线上** 。Fishhook 是按照下图的方式遍历 Mach-O 的多个段来找函数指针和函数符号名的映射关系，带来的 **副作用就是要大量的 Page In，对于大型 App 来说在 iPhone X 冷启耗时 200ms+** 。
[image:9E16CE4E-706F-444C-9AE4-6DC55E3E2772-1452-00007252854E8257/1d8520adcf0c43ceba0721d33405bec6~tplv-k3u1fbpfcp-zoom-1.image.png]
![[1d8520adcf0c43ceba0721d33405bec6~tplv-k3u1fbpfcp-zoom-1.image.png]]

如果不得不用 fishhook， **请在子线程调用，且不要在在`_dyld_register_func_for_add_image` 直接调用 fishhook** 。因为这个方法会持有 dyld 的一个全局互斥锁，主线程在启动的时候系统库经常会调用 `dlsym` 和 `dlopen` ，其内部也需要这个锁，造成上文提到的子线程阻塞主线程。

### 首帧渲染

不同 App 的业务形态不同，首帧渲染优化方式也相差的比较多，几个常见的优化点：

* LottieView：lottie 是 airbnb 用来做 AE 动画的库，但是加载动画的 json 和读图是比较慢的，可以 **先显示一帧静态图，启动结束后再开始动画，或者子线程预先把图和 json 设置到 lottie cache 里**
* Lazy 初始化 View： **不要先创建设置成 hidden，这是很不好的习惯**
* AutoLayout：AutoLayout 的耗时也是比较高的，但这块往往历史包袱比较重，可以 **评估 ROI 看看要不要改成 frame**
* Loading 动画：App 一般都会有个 loading 动画表示加载中，这个 **动画最好不要用 gif** ，线下测量一个 60 帧的 gif 加载耗时接近 70ms

### 其他 Tips

启动优化里有一些需要注意的 Tips：

**不要删除`tmp/com.apple.dyld` 目录** ，因为这个目录下存储着 iOS 13+ 的启动闭包，如果删除了下次启动会重新创建，创建闭包的过程是很慢的。接下来是 IO 优化，常见的方式是用 `mmap` 让 IO 更快一些，也可以在启动的早期预加载数据。

还有一些 iPhone 6 上耗时会明显增加的点：

* WebView User Agent：第一次在启动时获取，之后缓存，每次启动结束后刷新
* KeyChain：可以延迟获取或者预加载
* VolumeView：建议直接删掉

**iPhone 6 是个分水岭，性能会断崖式下跌，可以在 iPhone 6 上下掉部分用户交互来换取核心体验（记得 AB 验证）** 。

## Page In 耗时

启动路径上会触发大量 Page In，有没有办法优化这部分耗时呢？

### 段重命名

App Store 会对上传的 App 的 TEXT 段加密，在发生 Page In 的时候会解密，解密的过程是很耗时的。既然会 TEXT 段加密，那么直接的思路就是把 TEXT 段中的内容移动到其它段，ld 也有个参数 `rename_section` 支持重命名：
[image:AA313F2C-1A40-4E9B-8F55-329CF4F71D20-1452-0000725285BCE60F/7bc74ad2c3cb44b1896aecd26f75acdc~tplv-k3u1fbpfcp-zoom-1.image.png]
![[7bc74ad2c3cb44b1896aecd26f75acdc~tplv-k3u1fbpfcp-zoom-1.image.png]]

抖音重命名方案：

```
"-Wl,-rename_section,__TEXT,__cstring,__RODATA,__cstring",
"-Wl,-rename_section,__TEXT,__const,__RODATA,__const",
"-Wl,-rename_section,__TEXT,__gcc_except_tab,__RODATA,__gcc_except_tab",
"-Wl,-rename_section,__TEXT,__objc_methname,__RODATA,__objc_methname",
"-Wl,-rename_section,__TEXT,__objc_classname,__RODATA,__objc_classname",
"-Wl,-rename_section,__TEXT,__objc_methtype,__RODATA,__objc_methtype"
复制代码
```

这个优化方式在 iOS 13 下有效，因为 **iOS 13 优化了解密流程，Page In 的时候不需要解密了** ，这是 iOS 13 启动速度变快的原因之一。

### 二进制重排

既然启动的路径上会触发大量的 Page In，那么有没有什么办法优化呢？

启动具有局部性特征，即只有少部分函数在启动的时候用到，这些函数在中的分布是零散的，所以 Page In 读入的数据利用率并不高。如果我们可以把启动用到的函数排列到二进制的连续区间，那么就可以减少 Page In 的次数，从而优化启动时间：

以下图为例，方法 1 和方法 3 是启动的时候用到的，为了执行对应的代码，就需要两次 Page In。假如我们把方法 1 和 3 排列到一起，那么只需要一次 Page In，从而提升启动速度。
[image:E0044921-C7DE-4217-9BCE-AF0B0EC312E5-1452-00007252865E87A2/80b153b8b17848bda954cab6cb9ae93d~tplv-k3u1fbpfcp-zoom-1.image.png]
![[80b153b8b17848bda954cab6cb9ae93d~tplv-k3u1fbpfcp-zoom-1.image.png]]

链接器 ld 有个参数-order_file 支持按照符号的方式排列二进制。获取启动时候用到的符号主流有两种方式：

* 抖音方案：静态扫描获取 +load 和 C++静态初始化，hook objc_msgSend 获取 Objective C 符号。
* Facebook 方案：LLVM 函数插桩，灰度统计启动路径符号，用大多数用户的符号生成 order_file。

Facebook 的 LLVM 函数插桩是针对 order_file 定制，并且代码也是他们自己给 LLVM 开发的，目前已经合并到 LLVM 主分支了。
[image:0BA8DBAB-0D3E-42E8-8AA4-5C61CD07886E-1452-000072528520B59B/9e88d9aadde444689e8c7209b4fba1bf~tplv-k3u1fbpfcp-zoom-1.image.png]
![[9e88d9aadde444689e8c7209b4fba1bf~tplv-k3u1fbpfcp-zoom-1.image.png]]

Facebook 的方案更精细化，生成的 order_file 是最优解，但是工程量很大。抖音的方案 **不需要源码编译** ， **不需要对现有编译环境和流程改造** ，侵入性最小，缺点就是只能覆盖 90%左右的符号。

- 灰度是任何优化都要利用好的一个阶段，因为很多新的优化方案存在不确定性，需要先在灰度上验证。

## 非常规方案

### 动态库懒加载

最开始我们提到可以通过删代码的方式来减少代码量，那么有没有什么不减少代码总量，就可以减少启动时候要加载代码数量的方式呢？

* 答案就是动态库懒加载。

什么是懒加载的动态库呢？正常动态库都是会被主二进制直接或者间接链接的，那么这些动态库会在启动的时候加载。 **如果只打包进 App，不参与链接，那么启动的时候就不会自动加载，在运行时需要用到动态库里面的内容的时候，再手动懒加载** 。

懒加载动态库需要在编译期和运行时都进行改造，编译期的架构：
[image:253B6CDD-435B-41A3-8F63-B10394BC683B-1452-00007252853BED40/8367763a804c4519aa0116ccd84a6b78~tplv-k3u1fbpfcp-zoom-1.image.png]
![[8367763a804c4519aa0116ccd84a6b78~tplv-k3u1fbpfcp-zoom-1.image.png]]

像 A.framework 等动态库是懒加载的，因为并没有参与主二进制的直接 or 间接链接。动态库之间一定会有一些共同的依赖，把这些依赖打包成 Shared.framework 解决公共依赖的问题。

运行时通过 `-[NSBundle load]` 来加载，本质上调用的是底层的 `dlopen` 。那么什么时候触发动态库手动加载呢？

动态库可以分成两种：业务和功能。 **业务就是 UI 的入口，可以把动态库加载的逻辑收敛到路由内部，这样外部其实并不知道动态库是懒加载的，也能更好地容错** 。功能库（比如上图的 QR.framework）会有些不一样，因为没有 UI 等入口，需要功能库自己维护 Wrapper：
[image:0C2D32B8-C2E5-4064-B915-59F45556406E-1452-0000725284ED25F1/46a4509c9e1b448e9812baf47d3eb524~tplv-k3u1fbpfcp-zoom-1.image.png]
![[46a4509c9e1b448e9812baf47d3eb524~tplv-k3u1fbpfcp-zoom-1.image.png]]

* App 对 Wrapper 直接依赖，这样外部并不知道这个动态库是懒加载的
* Wrapper 内部封装了动态调用逻辑，动态调用指的是通过 dlsym 等方式调用

动态库懒加载除了启动加载的代码减少，还能长期防止业务增加代码引起启动劣化，因为业务的初始化在第一次访问的时候完成的。

这个方案还有其他优点，比如动态库化后本地编译时间会大幅度降低，对其他性能指标也有好处，缺点是会牺牲一定程度的包大小，但可以用段压缩等方式优化懒加载的动态库来打平这部分损耗。

### Background Fetch

Background Fetch 可以隔一段时间把 App 在后台启动，对于时间敏感的 App（比如新闻）可以在后台刷新数据，这样能够提高 Feed 加载的速度，进而提升用户体验。

那么，这种类似“后台保活”的机制，为什么能提高启动速度呢？我们来看一个典型的 case：
[image:EA14B744-3E3C-4C24-940B-CAB71DE933F5-1452-000072528687A759/5fdb6f6a52b146ed868a51824553b681~tplv-k3u1fbpfcp-zoom-1.image.png]
![[5fdb6f6a52b146ed868a51824553b681~tplv-k3u1fbpfcp-zoom-1.image.png]]

1. 系统在后台启动 App
2. 时间长因为内存等原因，后台的 App 被 kill 了
3. 这时候用户立刻启动 App，那么这次启动就是一次 **热启动** ，因为缓存还在
4. 又一次系统在后台启动 App
5. 这次用户在 App 在后台的时候点了 App，那么这次启动就是一次 **后台回前台** ，因为 App 仍然活着

通过这两个典型的场景，可以看出来为什么 Background Fetch 能提高启动速度了：

* **提高热启动在冷启动的占比**
* **后台启动回前台被定义为启动，因为用户的角度来说这就是一次启动**

后台启动有一些要注意的点， **比如日活，广告，甚至是 AB 进组逻辑都会受影响** ，需要做不少适配。往往需要启动器来支撑，因为正常启动在 didFinishLaunch 执行的任务，在后台启动的时候需要延迟到第一次回前台的时候再执行。

# 总结

最后提炼出几点我们认为在任何优化中都重要的：

* **白盒优化** ，知道为什么慢，优化的是哪部分。
* **线上数据都是优化的指南针** ，也是衡量优化效果的唯一方式，建议开 AB 实验，验证对业务上的影响。
* **不要忽略防劣化的建设** ，尤其是业务迭代迅速的团队，否则很有可能优化的速度赶不上劣化。
* **做长期的架构建设** ，良好的架构会长期为启动这些基础性能保驾护航。

[抖音品质建设 - iOS启动优化《实战篇》](https://juejin.cn/post/6921508850684133390)