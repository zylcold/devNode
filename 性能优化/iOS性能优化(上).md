#optimize 

## 1. 卡顿优化
### 了解CPU和GPU
在屏幕成像过程中，CPU和GPU的作用是至关重要的。
* **CPU**
- Central Processing Unit，中央处理器，在iOS程序中，负责对象的创建和销毁、对象属性的调整、布局的计算、文本的计算和排版规格、图片的格式转码和解码、图像的绘制（**Core Graphic**）
* **GPU** 
- Graphics Processing Unit，图形处理器，负责纹理的渲染。如果没有接触过OpenGL的朋友，可能不太好理解纹理渲染这个概念，我们知道，屏幕上面的物理元件是像素，我们在屏幕上面看到的图片，文字，视频，就是由屏幕上的所有像素，通过控制色值变化而呈现出来的。那么像素的色值数据，就是由GPU计算得出的，然后将这些数据提交给视频控制器，由它负责显示到屏幕上。

[image:25028FD5-9795-46BC-9D55-E2674BFB7547-439-000023FE32F40A3E/dfbcc46f8d564984b762349e40995506~tplv-k3u1fbpfcp-zoom-1.image.png]
![[dfbcc46f8d564984b762349e40995506~tplv-k3u1fbpfcp-zoom-1.image.png]]

iOS中采用的是双缓冲机制，分为前帧缓存和后帧缓存。

### 屏幕成像原理
屏幕成像原理是一个非常庞大的知识体系，这里仅介绍一下我们当前所需要了解的部分，以便我们接下来的话题。 屏幕的显示，是受控于两种信号
* **垂直同步信号（VSync）**
屏幕发出VSync之后，就表示将要进行新一帧画面的显示，于是开始从帧缓存里面读取经过GPU渲染好的用于显示的数据

* **水平同步信号（HSync）**
显示器从帧缓存里拿到数据之后，是从上到下一行一行的刷新的，刷新完一行，就发出一个HSync，直到最下面一层显示出来，这样，一帧的画面就完成了显示。

[image:3379B1B8-1503-4486-B013-5ADC325DC91E-439-000023FE32C2D906/7b29e97c3c344296b64dc654ae14244b~tplv-k3u1fbpfcp-zoom-1.image.png]
![[7b29e97c3c344296b64dc654ae14244b~tplv-k3u1fbpfcp-zoom-1.image.png]]

我们可以把屏幕想象成刷墙师傅，每一帧的数据就是桶里的油漆，而GPU就是负责提供油漆的店老板。

### 卡顿产生的原因
我们手机屏幕的刷帧率是60FPS（**Frame per Second** 帧/秒），也就是会所1秒钟的时间，屏幕可以刷新60帧（次）。完成一帧刷新的用时是16.6毫秒。因此垂直同步信号**VSync**就是每16.6毫秒发出一次。

两次**VSync**之间的这16.6毫秒，就是被CPU和GPU共同完成下一帧画面的计算和渲染工作的时间。但是CPU计算和GPU渲染所用的时间是取决于任务的运算量的，因此就有可能大于16.6毫秒，也有可能小于或者等于16.6毫秒。

这里我们假设Tc=CPU计算时间，Tg=GPU渲染时间。如果Tc+Tg <= 16.6ms，那么完美，下一帧画面的数据可以在**VSync**到来之间就准备好；但是如果Tc+Tg > 16.6ms，意味着屏幕将要开始显示下一帧画面了，但是CPU和GPU那里却还在咔咔咔的准备着画面数据，那么没办法，在接下来的**16.6ms**周期里面，屏幕就继续用上一帧的画面数据来显示。同一个画面被显示了过长的时间，就造成了视觉上可感知到的卡顿现象。再通过下图来体会一下
[image:C4EA216B-748C-4B98-BC6B-E6C89B840E21-439-000023FE32986C24/8fffc98e674c4da59bc9e67f7cd30473~tplv-k3u1fbpfcp-zoom-1.image.png]
![[8fffc98e674c4da59bc9e67f7cd30473~tplv-k3u1fbpfcp-zoom-1.image.png]]

上看我们了解了产生的原因，就是由于CPU计算时间和GPU的渲染时间过长导致的。因此想要优化卡顿问题，无非就是从CPU和GPU下手，减轻它们的工作量，以控制它们的操作耗时。

### 卡顿优化-CPU
首先开看看CPU，我们有如下途径来减轻它的计算任务

尽量用轻量级的对象，比如简单的数字，尽量选择基础数据类型，不要使用NSNumber，对象操作的开销肯定大于基础数据类型的开销。

CALayer是用来显示图像的，UIView是负责处理触摸交互事件的，UIView内部封装了CALayer属性，因此UIView的图像显示实际上是它内部的这个CALayer来完成的。因此如果我们不需要考虑触摸事件，只是单纯的要显示内容的话，可以考虑用CALayer取代UIView。

尽量提前计算好布局，在有需要的时候一次性调整对应属性，不要多次修改图片的size最好是跟UIImageView的size刚好一致，可以省去图片剪裁的操作开销控制一下线程的最大并发数

尽量把耗时操作放到子线程处理（比如文本的尺寸计算、绘制，图片的解码、绘制）

### 卡顿优化-GPU
对与GPU，有下列方案可以减少渲染开销
* 尽量避免段时间内大量的图片显示，尽可能将多张图片合成一张图片显示，比如说三张图片同时显示，不如将这三张图片合成到一块作为一张图片来显示

* GPU能处理的最大纹理尺寸是4096*4096，一旦超过这个尺寸，就会占用CPU资源进行处理，这样势必影响CPU的运算效率，因此纹理尽量不要超过这个尺寸

* 尽量减少视图的数量和层级

* 减少不必要的透明的视图

* 尽量避免离屏渲染

#### 离屏渲染
在OpenGL中，GPU有两种渲染方式
* **On-Screen Rendering**：当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作
* **Off-Screen Rendering**：离屏渲染，在当前屏幕缓冲区以外开辟一个新的缓冲区进行渲染操作

为什么离屏渲染消耗性能？
* 需要创建新的缓冲区
* 离屏渲染的整个过程中，需要多次切换上下文环境，先是从当前屏幕缓冲区（**On-Screen**）切换到离屏缓冲区(**Off-Screen**)，等完成离屏渲染操作之后，将离屏缓冲区的渲染结果显示到屏幕上，然后还需要将上下文环境从离屏缓冲区切换回当前屏幕缓冲区。

哪些操作会触发离屏渲染？
* 光栅化操作 layer.shouldRasterize = YES
* 遮罩设置 layer.mask
* 圆角设置 layer.masksToBounds = YES&layer.cornerRadius>0
🥝可以考虑通过CoreGraphics绘制剪裁圆角，或者叫美工提供圆角图片🥝
* 阴影设置 layer.shadowXXX
🥝如果设置了layer.shadowPath就不会产生离屏渲染🥝

### 卡顿检测
平时我们所碰到的卡顿，主要是在主线程执行了比较耗时的操作，可以在主线程**RunLoop**中添加observer，通过监听**RunLoop**的状态切换耗时，来监控卡顿。

##（二）耗电优化
iOS 设备耗电的主要来源有一下几种
[image:D4F25BEB-2247-4E5B-B49B-77399784795A-439-000023FE325C9ACF/3fbd1afde32a4aba8e16de4c60d27eab~tplv-k3u1fbpfcp-zoom-1.image.png]
![[3fbd1afde32a4aba8e16de4c60d27eab~tplv-k3u1fbpfcp-zoom-1.image.png]]

* CPU处理 **Processing**
* 网络 **Networking**
* 定位 **Location**
* 图像 **Graphics**


### 耗电优化方案
* 尽可能降低CPU、GPU功耗
* 少用定时器
* 优化I/O操作
	1. 尽量不要频繁的进行数据写入操作，最好批量一次性写入
	2. 读写大量重要数据时，考虑用dispatch_io，它提供了基于GCD的异步操作文件I/O的API。使用它时，系统会优化磁盘的访问
	3. 数据量比较大的话，建议使用数据库
* 网络优化
	1. 减少、压缩网络数据
	2. 如果多次请求的结果是相同的，尽量使用缓存
	3. 使用断点续传，否则网络不稳定时可能会导致多次传输相同内容
	4. 网络不可用时，不要尝试执行网络请求
	5. 让用户取消长时间运行或者速度很慢的网络操作，设置合适的超时时间
	6. 批量传输，比如，下载视频流是，不要传输很小的数据包，直接下载整个文件或者一大块一大块下载。如果下载广告，一次性多下载一些，然后慢慢的拿出来展示。如果狭隘电子邮件，就一次下载多封，不要一封一封下载
* 定位优化
	1. 如果只是需要确定用户的位置，最好用CLLocationManager的requestLocation方法。定位完成后，会自动让定位硬件断电。
	2. 如果不是导航引用，尽量不要实时更新位置，定位完毕就关闭掉定位服务
	3. 需要后台定位时，尽量设置pauseLocationUpdateAutomatically为YES，这样，如果用户不太可能移动的时候，系统就会自动暂停位置更新。
	4. 尽量不要使用startMonitoringSignificantLocationChanges，优先考虑startMonitoringForRegion:
* 硬件检测优化:
用户移动、摇晃、倾斜设备时，会产生动作事件（motion），这些事件是由加速度计、陀螺仪、磁力传感器等硬件检测的。在不需要检测的场合，应该及时关闭这些硬件。

### （三）启动优化
**App启动过程分析**
**App的启动可以分为两种**
* 冷启动：从零开始启动App
* 热启动：App已经在内存中，处在后台状态中，在次点击图标启动App
App的启动时间优化，主要是针对冷启动来进行优化的。那么首先我们就需要了解一下App的冷启动过程包含哪些步骤。
* （一）dyld阶段：dyld1dynamic link editor），Apple的动态链接器，可用来加载Mach-O文件（可执行文件、动态库等等）。冷启动一个app之后，首先是dyld开始工作，它负责两件事情：
	1. 加载App的可执行文件，同时会递归加载所有依赖库的动态库。
	2. 当完成可执行文件和动态库的加载之后，就通知Runtime进行下一步处理。
* （二）Runtime阶段：在这个阶段，Runtime做了如下的工作：
	1. 调用map_images函数对可执行文件的内容进行解析和处理。
	2. 在load_images函数中调用call_load_methods，以调用所有Class和Category的+load方法。
	3. 进行各种**objc**结构的初始化（例如**objc**类的注册，初始化类对象等等）
	4. 调用C++静态初始化器以及被_attribute_((constructor))修饰的函数
**到此为止，可执行文件和动态库中的所有所有**Symbols**（符号，包括 Class， Protocol， Selector， IMP等等）都已经按照规定格式加载到内存中，并且被runtime所管理**
* （三）main函数阶段
**小结**
* App的冷启动由dyld主导，将可执行文件加载到内存，顺便把所依赖的动态库也加载到内存，然后通知runtime进行相应处理
* runtime负责将上面的内容初始化成`**bjc**定义的结构体
* 最后当初始化工作完成后，dyld就会调用main函数。接下来便是main函数—>UIApplicationMain函数 —>didFinishLaunchingWithOptions方法。

[image:87831463-6DB7-4C69-8640-9154D0F08931-439-000023FE321A84E3/186f338fadad480ab914effa5971d13b~tplv-k3u1fbpfcp-zoom-1.image.png]
![[186f338fadad480ab914effa5971d13b~tplv-k3u1fbpfcp-zoom-1.image.png]]

**App启动优化**
**dyld阶段优化**
* 减少动态库，合并一些动态库，定期清理不必要的动态库
* 减少Objc类、Category的数量，减少Selector数量，定期清理不再需要的Class和Category
* 减少C++虚函数的数量，因为虚函数会导致额外的虚表的存在
* 如果是**Swift**尽量使用struct
**Runtime阶段优化**
* 用+initialize方法+dispatch_once的组合来取代所有的 _attribute_((constructor))、C++静态构造器、Objc的+load方法。
**main阶段优化**
在不影响用户体验的前提下，尽可能讲一些操作延迟，不要全都放在didFinishLaunchingWithOptions 方法中。 ###（四）安装包瘦身 iOS的安装包（ipa）主要有可执行文件和资源组成 **可执行文件**——就是由我们iOS项目的代码经过编译连接产生的二进制文件，要想对可执行文件进行瘦身，有如下几种思路：
* 编译器优化
Strip Linked Product、Make Strings Read-Only、Symbols Hidden by Default设置为YES 去掉异常支持，Enable C++ Exceptions、Enable Objective-C Exceptions设置为NO，Other C Flags添加-fno-exceptions
* 通过第三方工具 [AppCode](https://www.jetbrains.com/objc/) 检测项目中用不到的代码：菜单 —> **Code** —> **Inspect Code**
**资源**——包括我们iOS项目中的图片、音频、视频、**storyboard**等，对其进行优化的思路有
* 对资源进行无损压缩
* 去除没有用到的资源（ [第三方检测工具](https://github.com/tinymind/LSUnusedResources) ）

作者：RUNNING_NIUER
链接：https://juejin.cn/post/6966859086008680455