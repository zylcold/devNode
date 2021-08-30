#display  #optimize 

### 图像图形渲染原理

图形渲染主要是利用 `GPU` 并行运算能力，实现图形渲染并显示在屏幕的每一个像素上。渲染过程最常用的就是 *光栅化* ，即将数据转化为可见像素的过程。 `GPU` 及相关驱动实现了图形处理的 `OpenGL` 和 `DirectX` 模型，其实 `OpenGL` 不是函数API而是一种标准，制定了相关函数API及其实现的功能，具体的函数库由第三方来实现，通常是由显卡制造商来提供。

`GPU` 渲染过程如下图所示：

[image:CD893751-9898-432A-BBF1-6CDC32581874-21361-00056B3F3F913D95/17343b2854f2c174.jpeg]
![[17343b2854f2c174.jpeg]]

主要包括：顶点着色器(包含了3D坐标系的转换，每个顶点属性值设定)、形状(图元)装配(形成基本的图形)、几何着色器(构造新的顶点来形成其他形状，如上图的另一个三角形)、光栅化(将形状映射到屏幕的相应的像素生成 *片段* ，片段包含了像素结构所有的数据)、片段着色器(丢弃超过视图以外的像素并着色)、测试与混合(判断像素位置如是否在其他像素的后面及透明度等决定是否丢弃及混合)。

要想图形更加真实逼真需要更多的顶点及颜色属性，这样就增加了性能开销，为提升成产和执行效率，经常会使用 **纹理** 来表现细节。

> 纹理是一个 2D 图片（甚至也有 1D 和 3D 的纹理），纹理一般可以直接作为图形渲染流水线的*第五阶段(即片段着色器)*的输入；

`GPU` 内部包含了若干处理核来实现并发执行，其内部使用了二级缓存( `L1`、`L2` `cache` )，其与 `CPU` 的架构模型包含如下两种形式：分离式及耦合式，如下图所示：

[image:D56534B0-8F4F-41B0-9B78-33F479128098-21361-00056B3F3537D747/17343b2f4c973a2b.jpeg]
![[17343b2f4c973a2b.jpeg]]

* 分离式的结构

* CPU 和 GPU 拥有各自的存储系统，两者通过 PCI-e 总线进行连接。这种结构的缺点在于 PCI-e 相对于两者具有低带宽和高延迟，数据的传输成了其中的性能瓶颈。目前使用非常广泛，如PC、智能手机等。

* 耦合式的结构

* CPU 和 GPU 共享内存和缓存。AMD 的 APU 采用的就是这种结构，目前主要使用在游戏主机中，如 PS4。

屏幕图形显示结构如下：

[image:16103F18-0796-4E48-B621-0CB9C8A6BDA5-21361-00056B3F320A2B01/17343b34a17bc945.jpeg]
![[17343b34a17bc945.jpeg]]

`CPU` 将图形数据通过总线 `BUS` 提交至 `GPU` ， `GPU` 经过渲染处理转化为一帧帧的数据并提交至帧缓冲区，视频控制器会通过垂直同步信号 `VSync` 逐帧读取帧缓冲区的数据并提交至屏幕控制器最终显示在屏幕上。为解决一个帧缓冲区效率问题(读取和写入都是一个无法有效的并发处理)，采用 **双缓冲机制** ，在这种情况下，GPU 会预先渲染一帧放入一个缓冲区中，用于视频控制器的读取。当下一帧渲染完毕后，GPU 会直接把视频控制器的指针指向第二个缓冲器，如下图所示：

[image:F42D832E-083B-4EE7-8033-6CEC9945F138-21361-00056B3F31AF5ECF/17343b38634567f7.jpeg]
![[17343b38634567f7.jpeg]]

*双缓冲机制* 虽然提升了效率但也引入了 *画面撕裂* 问题，即当视频控制器还未读取完成时，即屏幕内容刚显示一半时，GPU 将新的一帧内容提交到帧缓冲区并把两个缓冲区进行交换后，视频控制器就会把新的一帧数据的下半段显示到屏幕上，造成画面撕裂现象，如下图：

[image:C94FE3BB-69A9-4210-BD55-35164C5F61C8-21361-00056B3F3157A0C8/17343b3bb0d12451.jpeg]
![[17343b3bb0d12451.jpeg]]

为了解决这个问题，GPU 通常有一个机制叫做 **垂直同步** （简写也是 V-Sync），当开启垂直同步后，GPU 会等待显示器的 VSync 信号发出后，才进行新的一帧渲染和缓冲区更新。这样能解决画面撕裂现象，也增加了画面流畅度，但需要消费更多的计算资源，也会带来部分延迟。

> iOS 设备会始终使用双缓存，并开启垂直同步。而安卓设备直到 4.1 版本，Google 才开始引入这种机制，目前安卓系统是三缓存+垂直同步。

#### 卡顿

[image:2E8E8DCF-B4FF-45EA-A143-667CE02E7A31-21361-00056B3F31113B11/17343b413b245368.jpeg]
![[17343b413b245368.jpeg]]

在 `VSync` 信号到来后，系统图形服务会通过 `CADisplayLink` 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次 `VSync` 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 `VSync` 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。

### 图像显示

#### 图形渲染技术栈

整个图形渲染技术栈：App 使用 `Core Graphics`、`Core Animation`、`Core Image` 等框架来绘制可视化内容，这些软件框架相互之间也有着依赖关系。这些框架都需要通过 `OpenGL` 来调用 GPU 进行绘制，最终将内容显示到屏幕之上，结构如下图所示：

[image:C3DC52E5-C6D9-4FA6-868D-7FEEEC971101-21361-00056B3F30AF6BDA/17343b442ad11c23.jpeg]
![[17343b442ad11c23.jpeg]]

框架介绍：

* UIKit

* `UIKit` 自身并不具备在屏幕成像的能力，其主要负责对 **用户操作事件的响应** （ `UIView` 继承自 `UIResponder` ），事件响应的传递大体是经过逐层的 **视图树** 遍历实现的。

* Core Animation

* `Core Animation` 是一个复合引擎，其职责是 **尽可能快地组合屏幕上不同的可视内容，这些可视内容可被分解成独立的图层（即 CALayer），这些图层会被存储在一个叫做图层树的体系之中** 。从本质上而言， `CALayer` 是用户所能在屏幕上看见的一切的基础。

* Core Graphics

* `Core Graphics` 基于 Quartz 高级绘图引擎，主要用于 **运行时绘制图像** 。开发者可以使用此框架来处理基于路径的绘图，转换，颜色管理，离屏渲染，图案，渐变和阴影，图像数据管理，图像创建和图像遮罩以及 PDF 文档创建，显示和分析。

* Core Image

* `Core Image` 与 `Core Graphics` 恰恰相反， `Core Graphics` 用于 **在运行时创建图像** ，而 `Core Image` 是用来处理 **运行前创建的图像** 的。 `Core Image` 框架拥有一系列现成的图像过滤器，能对已存在的图像进行高效的处理。

* OpenGL(ES)

* `OpenGL ES` （OpenGL for Embedded Systems，简称 GLES），是 OpenGL 的子集。

* Metal

* `Metal` 类似于 `OpenGL ES` ，也是一套 **第三方标准** ，具体实现 *由苹果实现* 。大多数开发者都没有直接使用过 `Metal` ，但其实所有开发者都在间接地使用 `Metal` 。 `Core Animation`、`Core Image`、`SceneKit`、`SpriteKit` 等等渲染框架都是构建于 `Metal` 之上的。当在真机上调试 OpenGL 程序时，控制台会打印出启用 `Metal` 的日志。根据这一点可以猜测， *Apple 已经实现了一套机制将 OpenGL 命令无缝桥接到 `Metal` 上，由 `Metal`担任真正于硬件交互的工作* 。

#### UIView与CALayer关系

`UIKit` 中的每一个视图控件其内部都有一个关联的 `CALayer` ，即 `backing layer` ；由于这种一一对应的关系，视图采用 **视图树** 形式呈现，与之对应的图层也是采用 **图层树** 形式。

> 视图的职责是创建并管理图层，以确保当子视图在层级关系中 **添加或被移除** 时， **其关联的图层在图层树中也有相同的操作** ，即保证视图树和图层树在结构上的一致性。

苹果采用这种结构的目的是保证iOS/Mac平台底层 `CALayer` 通用，避免重复代码且职责分离，毕竟采用多点触摸形式与基于鼠标键盘的交互有着本质的区别；

#### CALayer

`CALayer` 基本等同于 **纹理** ，本质上是一张图片，因此 `CALayer` 也包含一个 `contents` 属性指向一块缓存区，称为 `backing store` ，可以存放位图（Bitmap）。iOS 中将该缓存区保存的图片称为 **寄宿图** 。

> **位图** （英语：Bitmap，台湾称为 **点阵图** ），又称 **栅格图** （Raster graphics），是使用 [像素](https://zh.wikipedia.org/wiki/%E5%83%8F%E7%B4%A0) [阵列](https://zh.wikipedia.org/wiki/%E9%99%A3%E5%88%97) (Pixel-array/Dot-matrix [点阵](https://zh.wikipedia.org/wiki/%E7%82%B9%E9%98%B5) )来表示的 [图像](https://zh.wikipedia.org/wiki/%E5%9B%BE%E5%83%8F) 。位图也可指：一种数据结构，代表了有限域中的稠集（dense set），每一个元素至少出现一次，没有其他的数据和元素相关联。在索引，数据压缩等方面有广泛应用，位图的像素都分配有特定的位置和 [颜色](https://zh.wikipedia.org/wiki/%E9%A2%9C%E8%89%B2) 值。

[image:EEFDCC79-F4F2-4100-801A-880725AC7F35-21361-00056B3F3065F7BE/17343b4a347d364f.jpeg]
![[17343b4a347d364f.jpeg]]

图形渲染流水线支持从顶点开始进行绘制（在流水线中，顶点会被处理生成纹理），也支持直接使用纹理（图片）进行渲染。相应地，在实际开发中，绘制界面也有两种方式：一种是 **手动绘制(custom drawing)** ；另一种是 **使用图片(contents image)** 。

`Contents Image` 是指通过 `CALayer` 的 `contents` 属性来配置图片，典型的是通过 `CGImage` 来指定其内容。 `Custom Drawing` 是指使用 `Core Graphics` 直接绘制寄宿图。实际开发中，一般通过继承 `UIView` 并实现 `-drawRect:` 方法来自定义绘制。

虽然 `-drawRect:` 是一个 `UIView` 方法，但事实上都是底层的 `CALayer` 完成了重绘工作并保存了产生的图片。下图所示为 `-drawRect:` 绘制定义寄宿图的基本原理。

[image:DC90058A-649F-4CB8-AFF9-F4F73D384964-21361-00056B3F2F6E4498/17343b4dd800eb37.jpeg]
![[17343b4dd800eb37.jpeg]]

* `UIView` 都有一个 `CALayer` 属性

* `CALayer` 存在弱引用 `delegate` 属性，实现了 `<CALayerDelegate>协议` ，由 `UIView` 来代理实现协议方法；

* 当需要重绘时， `CALayer` 首先调用 `-displayLayer` 方法，此时代理可以直接设置 `contents` 属性；

* 需要重绘指：比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法；

* 如果代理没有实现 `-displayLayer:` 方法， `CALayer` 则会尝试调用 `-drawLayer:inContext:` 方法。在调用该方法前， `CALayer` 会创建一个空的寄宿图（尺寸由 `bounds` 和 `contentScale` 决定）和一个 `Core Graphics` 的绘制上下文 `CGContextRef` ，为绘制寄宿图做准备，作为 `ctx` 参数传入。

* `-drawLayer:inContext` 内部会调用 `-drawRect` ，细节代码如下：

```
- (void)drawLayer:(CALayer*)layer inContext:(CGContextRef)context {
    UIGraphicsPushContext(context);

    CGRect bounds;
    bounds = CGContextGetClipBoundingBox(context);
    [self drawRect:bounds];

    UIGraphicsPopContext();
}
复制代码
```

* 具体的函数调用栈如下：

[image:644F9733-4F23-42DF-90B7-0F03824B6FAF-21361-00056B3F2E32D877/17343b52303b6707.jpeg]
![[17343b52303b6707.jpeg]]

* 最后，由 `Core Graphics` 绘制生成的寄宿图会存入 `backing store` 。

#### Core Animation Pipeline

了解完 `CALayer` 本质及流程后，详细介绍下 `Core Animation Pipeline` 工作原理，如下图：

[image:2AD7A880-81C5-4AA0-9A11-70460CCE4603-21361-00056B3F2DF2FBF4/17343b5513fa0cbe.jpeg]
![[17343b5513fa0cbe.jpeg]]

其中iOS中应用并不负责渲染而是由专门的渲染进程负责，即 `Render Server` ；

> 在 iOS 5 以前这个进程叫 SpringBoard，在 iOS 6 之后叫 BackBoard或者backboardd；

> 越狱查看系统进程，确实存在此进程，如下图：

[image:1D6E0D91-2010-4737-B42B-0E274233B376-21361-00056B3F2D9EB454/17343b58941c11ac.jpeg]
![[17343b58941c11ac.jpeg]]

主要处理流程如下：

* 首先，由 App 处理事件（Handle Events），如：用户的点击操作，在此过程中 app 可能需要更新 **视图树** ，相应地， **图层树** 也会被更新；

* 其次，App 通过 CPU 完成对显示内容的计算，如：视图的创建、布局计算、图片解码、文本绘制等。在完成对显示内容的计算之后，App 对图层进行打包，并在下一次 `RunLoop` 时将其发送至 `Render Server` ，即完成了一次 `Commit Transaction` 操作。

* 具体 `commit transcation` 可以细分为如下步骤：

	* `Layout` ，主要进行视图构建，包括： `LayoutSubviews` 方法的重载， `addSubview:` 方法填充子视图等；
	* `Display` ，主要进行视图绘制，这里仅仅是设置最要成像的图元数据。重载视图的 `drawRect:` 方法可以自定义 `UIView` 的显示，其原理是在 `drawRect:` 方法内部绘制寄宿图，该过程使用 CPU 和内存；
	* `Prepare` ，属于附加步骤，一般处理图像的解码和转换等操作；
	* `Commit` ，主要将图层打包，并将它们通过IPC发送至 `Render Server` 。该过程会递归执行，因为图层和视图都是以树形结构存在。

* `Render Server` 执行 `OpenGL`、`Core Graphics` 相关操作，如根据 `layer` 的各种属性(如果是动画属性，则会计算动画 `layer` 的属性的中间值)并用 `OpenGL` 准备渲染；

* `GPU` 通过 `Frame Buffer`、视频控制器等相关组件对图层进行渲染到屏幕；

为了满足屏幕60FPS刷新率， `RunLoop` 每次操作的时间间隔不应超过16.67ms，且上述步骤需要并行执行。

#### 渲染与RunLoop

[image:CEF5AED2-2BD1-4766-98C7-BE104E03609D-21361-00056B3F2B54D45B/17343b5c71858258.jpeg]
![[17343b5c71858258.jpeg]]

iOS 的显示系统是由 `VSync` 信号驱动的， `VSync` 信号由硬件时钟生成，每秒钟发出 60 次（这个值取决设备硬件，比如 iPhone 真机上通常是 59.97）。iOS 图形服务接收到 `VSync` 信号后，会通过 IPC 通知到 App 内。App 的 `Runloop` 在启动后会注册对应的 `CFRunLoopSource` 通过 `mach_port` 接收传过来的时钟信号通知，随后 `Source` 的回调会驱动整个 App 的动画与显示。

> 备注：实际观察App启动后未注册相关的 `VSync` 相关的 `Source` ，因此上述应用应该是 `Render Server` 渲染进程注册 `Source` 监听 `VSync` 信号来驱动图层的渲染，进而提交至GPU。

`Core Animation` 在 `RunLoop` 中注册了一个 `Observer` ，监听了 `BeforeWaiting` 和 `Exit` 事件。这个 Observer 的优先级是 2000000，低于常见的其他 Observer。当一个触摸事件到来时， `RunLoop` 被唤醒，App 中的代码会执行一些操作，比如创建和调整视图层级、设置 UIView 的 frame、修改 CALayer 的透明度、为视图添加一个动画；这些操作最终都会被 `CALayer` 捕获，并通过 `CATransaction` 提交到一个中间状态去（ `CATransaction` 的文档略有提到这些内容，但并不完整）。当上面所有操作结束后， `RunLoop` 即将进入休眠（或者退出）时，关注该事件的 `Observer` 都会得到通知。这时 `Core Animation` 注册的那个 `Observer` 就会在回调中，把所有的中间状态合并提交到 GPU 去显示；如果此处有动画， `Core Animation` 会通过 `CADisplayLink` 等机制多次触发相关流程。

### 渲染性能优化

为了保证渲染性能，主要是保证 `CPU` 及 `GPU` 不会阻碍上述渲染流程进而引发“掉帧”现象，因此需要分别针对 `CPU` 及 `GPU` 影响渲染过程进行分析、评估及优化。

#### CPU资源消耗原因及解决方案

##### 对象创建

对象创建会分配内存、调整属性、甚至还有读取文件(如创建 `UIViewController` 读取 `xib` 文件)等操作，比较消耗CPU资源。因此，尽量使用轻量的对象替代重量的对象，如 `CALayer` 比 `UIView` 不需要响应触摸事件；如果对象不涉及UI操作，则尽量放到后台线程执行；性能敏感的视图对象，尽量使用代码创建而不是 `Storyboard` 来创建；如果对象可以复用，可以使用缓存池来复用。

##### 对象调整

对象调整也经常是消耗CPU资源的地方，如 `CALayer` 属性修改、视图层次调整、添加和移除视图等；

> `CALayer` 内部并没有属性方法，其内部是通过 `runtime` 动态接收方法 `resoleInstanceMethod` 方法为对象临时添加一个方法，并把对应属性值保存到内部的一个 `Dictionary` 字典里，同时还会通知 `delegate`、创建动画等。 `UIView` 的关于显示相关的属性(比如 `frame/bounds/transform` )等实际上是 `CALayer` 属性映射来的。

##### 对象销毁

虽然对象销毁销毁资源不多，但累积起来也不容忽视。通常当容器类持有大量对象时，其销毁时的资源消耗就非常明显，因此，可见用于后台线程去释放的对象挪动后台线程去。技巧代码如下：

```
//将对象捕获到block中，然后扔到后台队列中随便发个消息以避免编译器警告；
NSArray *tmp = self.array;
self.array = nil;
dispatch_async(queue, ^{
    [tmp class];
});
复制代码
```

##### 布局计算

视图布局计算是应用最为常见的销毁CPU资源的地方，其最终实现都会通过 `UIView.frame/bounds/center` 等属性的调整上，因此，避免CPU资源消耗尽量提前计算好布局，在需要时一次性调整好对应属性，而不要多次、频繁的计算和调整这些属性。

##### Autolayout

`Auotlayout` 是苹果提倡的技术，可大部分情况下能很好地提升开发效率，但是其对于复杂视图来说尝尝会带来严重的性能问题，具体可参阅 [pilky.me/36/](http://pilky.me/36/) ，因此对于性能要求高的视图尽量使用代码实现视图。

##### 文本计算

如果页面包含大量文本，文本宽高计算会占用很大一部分资源，并且不可避免。可以通过 `UILabel` 内部的实现方式： `[NSAttributedString boundingRectWithSize:options:context]` 富文本 `AttributedString` 来计算文本宽高，用 `[NSAttributeString drawWithRect:options:context:]` 来绘制文本，并放在后台线程执行避免阻塞主线程；或者使用 `CoreText` 基于c的跨平台API来绘制文本。

> Core Text 是为一些必须处理底层字体处理和文字布局的开发者准备，如无必要，你应该使用 TextKit（ [Text Programming Guide for iOS](https://developer.apple.com/library/archive/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009542) ）、CocoaText（ [Cocoa Text Architecture Guide](https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009459) ）等框架开发你的 App 或 Mac 应用。Core Text 是以上两种文本框架的底层实现，因此它们的速度和效率是共享的。除此之外，以上两种文本框架提供了富文本编辑及页面布局引擎。如果你的 App 只使用 Core Text，则需要为其提供其他的基础实现。 [Core Text 编程指南](https://juejin.im/post/5c5154e9e51d4503834dabf4)

##### 文本渲染

屏幕上能看到的所有文本内容控件，包括 `UIWebView` ，在底层都是通过 `CoreText` 排版、绘制为 `Bitmap` 显示。常见的文本控件，如 `UILabel`、`UITextView` 等，其排版和绘制都是在主线程进行，当显示大量文本时，CPU的压力会非常大。解决方案只有一个，就是自定义文本控件，并用 `TextKit` 或最底层的 `CoreText` 对文本 *异步绘制* 。

##### 图片解码

当使用 `UIImage` 或 `CGImageSource` 的那几个方法创建图片时，图片数据并不会立即解码。只有图片设置到 `UIImageView` 或者 `CALayer.contents` 中去，并且 `CALayer` 被提交到GPU前， `CGImage` 中的数据才会得到解码，且需要在主线程执行。

> 解决方法：后台线程先把图片绘制到 `CGBitmapContext` 中，然后从 `Bitmap` 直接创建图片。目前常见的网络图片库都自带这个功能。

##### 图像绘制

图像的绘制通常是指用 `CGxx` 开头的方法将图像绘制到画布中，然后从画布创建图片并显示这样的一个过程。这个最常见的就是 `[UIView drawRect:]` 方法，由于 `CoreCraphic` 方法通常都是线程安全的，所以图像的绘制可以很容易放到后台线程进行，示例如下：

```
- (void)display {
	dispatch_async(backgroudQueu, ^{
		CGContextRef ctx = CGBitmapContextCreate(...);
		//draw in context ....
		CGImageRef img = CGBitmapContextCreateImage(ctx);
		CFRelease(ctx);
		dispatch_async(mainQueue, ^{
			layer.contents = img;
		});
	});
}
复制代码
```

#### GPU资源消耗原因及解决方案

相对于CPU来说，GPU主要就是：接收提交的 *纹理* 和 *顶点描述(三角形)* ， *应用变换*、*混合* 并 *渲染* ，然后输出到屏幕上。通常你所能看到的内容，主要也就是纹理(图片)和形状(三角模拟的矢量图形)两类。

##### 纹理的渲染

所有的 `Bitmap` ，包括图片、文本、栅格化的内容，最终都要从内存提交到显存，绑定为GPU纹理。不论是提交到显存的过程，还是GPU调制和渲染纹理的过程，都要消耗不少GPU资源。当在较短时间内显示大量图片时(如 `UITableView` 存在非常多的图片并且快速滑动时)，CPU占用率很低，GPU占用非常高，因此会导致界面掉帧卡顿。有效避免此情况的方法就是尽量减少在短时间内大量图片的显示，尽可能将多张图片合并为一张进行显示。

##### 视图的混合

多存在多视图且多层次重叠显示时，GPU会首先将其混合在一起。如果视图结构很复杂，混合的过程也会消耗很多的GPU资源。为了减轻GPU的消耗，应尽量减少视图数量级层次，并在不透明的视图里标明 `opaque` 属性以避免无用的 `Alpha` 通道合成。

##### 图形的生成

`CALayer` 的 `border`、圆角、阴影、遮罩( `mask` )， `CASharpLayer` 的矢量图形显示，通常会触发离屏渲染( `offscreen rendering` )，而离屏渲染通常发生在GPU中。当一个列表视图中存在大量圆角的 `CALayer` 且款式滑动时，会消耗大量的GPU资源，进而引发界面卡顿。为避免此种情况，可以尝试开始 `CALayer.shouldRasterize` 属性，这会吧离屏渲染的操作转嫁到CPU上；最好是尽量避免使用圆角、阴影、遮罩等属性。

> GPU屏幕渲染存在两种方式： *当前屏幕渲染(On-Screen Rendering)*和* 离屏渲染(Off-Screen Rendering)* ，其中当前屏幕渲染就是正常的GPU渲染流程，GPU将渲染完成的帧放到帧缓冲区，然后显示到屏幕；而离屏渲染会额外创建一个离屏渲染缓冲区(如保存后续复用的数据)，后续仍会提交至帧缓冲区进而显示到屏幕。

> 离屏渲染需要创建新的缓冲区，渲染过程中会涉及从当前屏幕切换到离屏环境多次上下文环境切换，等到离屏渲染完成后还需要将渲染结果切换到当前屏幕环境，因此付出的代价较高。

#### AsyncDisplayKit

`AsyncDisplayKit` (简写 `ASDK` )是Facebook开源的一个用于保持iOS界面流畅的开源库，其基本原理如下：

[image:0FC17321-B0B6-45E8-8CB4-92CEC64D8CA0-21361-00056B3F2B0BDAF6/17343b64cea66951.jpeg]
![[17343b64cea66951.jpeg]]

将不需要主线程执行的消耗性能的通过异步执行方式执行，如文本宽高和视图布局计算，文本渲染、图片界面和图形绘制，对象创建、属性调制和销毁；但 `UIKit` 和 `Core Animation` 相关操作必须在主线程执行，对于不能后台执行的就优化性能。

##### UIView CALayer封装

[image:DDCA2DE2-943C-43F7-B494-50717DD8315A-21361-00056B3F2AC2447F/17343b66fee182d9.jpeg]
![[17343b66fee182d9.jpeg]]

在原有 `UIView` 和 `CALayer` 基础上，封装了 `ASDisplayNode` 类(简写 `ASNode` )，包装了常见的视图属性(如 `frame/bounds/aplphs/transform/backgroudColor/superNode/subNodes` 等)，建立 `ASNode` 与 `CALayer` 的对应关系，当 `CALayer` 属性改变或者动画产生时，会通过 `delegate` 通知的 `UIVIew` 进而通知 `ASNode` 。由于 `UIview` 和 `CALayer` 不是线程安全的，并且只能在主线程创建、访问和销毁，但 `ASNode` 是线程安全的，可以在后台线程创建和修改。 `ASNode` 还提供了 `layer backed` 属性，当不需要触摸事件时，就省去了 `UIView` 的中间层功能。同时还提供了大量优化后的子类封装，如 `Button/Control/Cell/Image/ImageView/Text/TableView/CollectView` 等。

##### 图层预合成

对于多层级 `CALayer` 情况，GPU需要图层合成，但对于多层级图层中不需要动画和位置调整的情况，就会导致没必要的GPU性能消耗，因此 `ASDK` 为此实现了一个 `pr-composing` 的技术，将多层级图层合并渲染成一张图片，有效降低了GPU的消耗。

##### 异步并发操作

上文提到的可以后台线程的任务通过GCD异步并发执行，有效利用iPhone处理器多核的特点。

##### RunLoop任务分发

ASDK 在此处模拟了 Core Animation 的这个机制：所有针对 ASNode 的修改和提交，总有些任务是必需放入主线程执行的。当出现这种任务时，ASNode 会把任务用 ASAsyncTransaction(Group) 封装并提交到一个全局的容器去。ASDK 也在 RunLoop 中注册了一个 Observer，监视的事件和 CA 一样，但优先级比 CA 要低。当 RunLoop 进入休眠前、CA 处理完事件后，ASDK 就会执行该 loop 内提交的所有任务。通过这种机制，ASDK 可以在合适的机会把异步、并发的操作同步到主线程去，并且能获得不错的性能。

### 卡顿检测

#### instrument工具

主要工具使用如下：

[image:B680A50D-DE25-45F7-B1F4-13B422D8326A-21361-00056B3F2A7C2115/17343b6e6af5f4bf.jpeg]
![[17343b6e6af5f4bf.jpeg]]

> **Time Profiler** ，用来检测CPU的使用情况。它可以告诉我们程序中的哪个方法正在消耗大量的CPU时间。使用大量的CPU并 *不一定* 是个问题 - 你可能期望动画路径对CPU非常依赖，因为动画往往是iOS设备中最苛刻的任务。 但是如果你有性能问题，查看CPU时间对于判断性能是不是和CPU相关，以及定位到函数都很有帮助。

> **Core Animation** ，用来监测Core Animation性能。它给我们提供了周期性的FPS。

如下图使用 `Core Animation` 工具查看 `FPS(Frames Per Second)` 每秒帧渲染数；

[image:65827813-F735-4D11-A03B-B13DFDBC1661-21361-00056B3F27E2D3A7/17343b720b5d62e2.jpeg]
![[17343b720b5d62e2.jpeg]]

#### 基于RunLoop检测

主要有两种方案：

* FPS监控

* 原理就是添加 `CADisplayLink` 对象至 `runloop` 中统计每秒回调次数，通过次数/时间来获取屏幕刷新率 `FPS` ，具体实现如下：

```
// 创建CADisplayLink，并添加到当前run loop的NSRunLoopCommonModes
_link = [CADisplayLink displayLinkWithTarget:self selector:@selector(tick:)];
[_link addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];

- (void)tick:(CADisplayLink *)link {
    if (_lastTime == 0) {
        _lastTime = link.timestamp;
        return;
    }
    
    _count++;
    NSTimeInterval delta = link.timestamp - _lastTime;
  	// 统计每秒的回调次数_count
    if (delta < 1) return;
    _lastTime = link.timestamp;
  	// FPS=次数/时间间隔
    float fps = _count / delta;
    _count = 0;    
    NSLog(@"current FPS: %d", (int)round(fps));
}
复制代码
```

> `CADisplayLink` 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 `NSTimer` 并不一样，其内部实际是操作了一个 `Source` ）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 `NSTimer` 相似），造成界面卡顿的感觉。在快速滑动 `TableView` 时，即使一帧的卡顿也会让用户有所察觉。通过对比 `CADisplayLink` 添加至 `runloop` 前后 `modes` 变化，发现其实现是向 `runloop` 中添加 `Source1` 回调为 `IODispatchCalloutFromCFMessage` ；

UI绘制并不一定 `FPS` 为满60帧，如动画片FPS为24，因此，通过 `FPs` 方案监测卡顿是存在问题的。

> FPS 是一秒显示的帧数，也就是一秒内画面变化数量。如果按照动画片来说，动画片的 FPS 就是 24，是达不到 60 满帧的。也就是说，对于动画片来说，24 帧时虽然没有 60 帧时流畅，但也已经是连贯的了，所以并不能说 24 帧时就算是卡住了。

* 主线程卡顿监控

* 通过子线程监测主线程的 `runloop` ，判断 `kCFRunLoopBeforeSources` 和 `kCFRunLoopAfterWaiting` 两个状态之间的耗时是否达到一定阈值，若监测到卡顿则记录此时的函数调用信息，具体代码如下：

```
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    MyClass *object = (__bridge MyClass*)info;
    
    // 记录状态值
    object->activity = activity;
    
    // 发送信号
    dispatch_semaphore_t semaphore = moniotr->semaphore;
    dispatch_semaphore_signal(semaphore);
}

- (void)registerObserver
{
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                            kCFRunLoopAllActivities,
                                                            YES,
                                                            0,
                                                            &runLoopObserverCallBack,
                                                            &context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    
    // 创建信号
    semaphore = dispatch_semaphore_create(0);
    
    // 在子线程监控时长
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES)
        {
            // 假定连续5次超时50ms认为卡顿(当然也包含了单次超时250ms)
            long st = dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 50*NSEC_PER_MSEC));
            if (st != 0)
            {
                if (activity==kCFRunLoopBeforeSources || 
                    activity==kCFRunLoopAfterWaiting)
                {
                    if (++timeoutCount < 5)
                        continue;
                    //使用第三方crash收集库PLCrashReporter，其不仅会收集crash信息也可以用于实施获取各线程的调用堆栈
                  	PLCrashReporterConfig *config = [[PLCrashReporterConfig alloc]
                                                   initWithSignalHandlerType:PLCrashReporterSignalHandlerTypeBSD                                  
                                                   symbolicationStrategy:PLCrashReporterSymbolicationStrategyAll];
                    PLCrashReporter *crashReporter = [[PLCrashReporter alloc] initWithConfiguration:config];
                    
                    NSData *data = [crashReporter generateLiveReport];
                    PLCrashReport *reporter = [[PLCrashReport alloc] initWithData:data error:NULL];
                    NSString *report = [PLCrashReportTextFormatter stringValueForCrashReport:reporter
                                                                              withTextFormat:PLCrashReportTextFormatiOS];

                    NSLog(@"好像有点儿卡哦");
                }
            }
            timeoutCount = 0;
        }
    });
}
复制代码
```

* 为啥需要监测 `kCFRunLoopBeforeSources` 和 `kCFRunLoopAfterWaiting` 间的耗时，主要因为两者之间处理了APP内部事件处理的 `Source0` 时间，如触摸事件、`CFSocketRef` ，还有中间监听 `kCFRunLoopBeforeWaiting` 状态 `Core Animation` 提交所有的图层中间状态至GPU，大部分导致卡顿的场景都在这两者之间；

* 而主线程 `RunLoop` 闲置时处在 `kCFRunLoopBeforeSources` 和 `kCFRunLoopAfterWaiting` 之间的 `kCFRunLoopBeforeWaiting` 状态，因此导致错误的判断为卡顿，因此优化解决此问题出现了 *子线程ping* 方案。具体的原理如下：创建一个子线程通过信号量去ping主线程，因为ping的时候主线程肯定是在 `kCFRunLoopBeforeSources` 和 `kCFRunLoopAfterWaiting` 之间。每次检测时设置标记位为YES，然后派发任务到主线程中将标记位设置为NO。接着子线程沉睡超时阙值时长，判断标志位是否成功设置成NO，如果没有说明主线程发生了卡顿， [ANREye](https://github.com/zixun/ANREye) 中就是使用子线程Ping的方式监测卡顿的，具体代码如下：

```
@interface PingThread : NSThread
......
@end

@implementation PingThread

- (void)main {
    [self pingMainThread];
}

- (void)pingMainThread {
    while (!self.cancelled) {
        @autoreleasepool {
          __block BOOL timeOut = YES;
            dispatch_async(dispatch_get_main_queue(), ^{
              	timeOut = NO;
              	dispatch_semaphore_signal(_semaphore);
            });
            [NSThread sleepForTimeInterval: lsl_time_out_interval];
            if (timeOut) {
                NSArray *callSymbols = [StackBacktrace backtraceMainThread];
              	...
            }
            dispatch_wait(_semaphore, DISPATCH_TIME_FOREVER);
        }
    }
}
@end
复制代码
```

### Reference

[iOS-Core-Animation-Advanced-Techniques](https://github.com/qunten/iOS-Core-Animation-Advanced-Techniques)
[计算机那些事(8)——图形图像渲染原理](http://chuquan.me/2018/08/26/graphics-rending-principle-gpu/)
[iOS 图像渲染原理](http://chuquan.me/2018/09/25/ios-graphics-render-principle/)
[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
[关于drawRect](https://www.jianshu.com/p/c49833c04362)
[深入理解 iOS Rendering Process](https://lision.me/ios-rendering-process/)
[iOS 视图、动画渲染机制探究](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&amp;mid=402348480&amp;idx=1&amp;sn=7f737a96b9a9e7baad12b48ebc6b1efe&amp;scene=0&amp;key=ac89cba618d2d9765dea8e02c0365bc4a490199938ebd108346b0361994543a6ae2a4655870436a42627968d2beac841&amp;ascene=0&amp;uin=MjAyNzY1NTU%3D&amp;devicetype=iMac+MacBookPro12%2C1+OSX+OSX+10.11.1+build%2815B42%29&amp;version=11020201&amp;pass_ticket=YYDGAMjGcYJJlj%2Bh72BXctaqS6yuDJlVNZ6LhIpUFMc%3D)
[iOS Core Animation: Advanced Techniques中文译本](https://zsisme.gitbooks.io/ios-/content/)
[iOS离屏渲染](https://juejin.im/post/5f0446f9f265da22d95da35a)
[iOS 的离屏渲染](https://juejin.im/entry/58c75b061b69e6006bea6827)
[离屏渲染优化详解：实例示范+性能测试](https://www.jianshu.com/p/ca51c9d3575b)
[iOS 核心动画高级及技巧](https://zsisme.gitbooks.io/ios-/content/chapter12/instruments.html)
[iOS性能优化 - 工具Instruments之Time Profiler](https://juejin.im/post/5b611766f265da0f800df59a)
[CADisplayLink](https://developer.apple.com/documentation/quartzcore/cadisplaylink)
[backboardd](http://iphonedevwiki.net/index.php/Backboardd)
[质量监控-卡顿检测](https://juejin.im/post/5bb09795f265da0ac84946e0)
[iOS开发--APP性能检测方案汇总(一)](https://www.jianshu.com/p/95df83780c8f)
[一文读懂iOS图像显示原理与优化 - 掘金](https://juejin.im/post/5f0b2efff265da230e6b62fb)