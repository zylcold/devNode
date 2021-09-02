## 前言
为了防止一个应用占用过多的系统资源，苹果设计了一个“看门狗”( `watchdog` )的机制。在不同的场景下，“看门狗”会监测应用的性能。如果超出了该场景所规定的运行时间，“看门狗”就会强制终结这个应用的进程。开发者们在 `crashlog` 里面，会看到诸如 `0x8badf00d` 这样的错误代码。异常代码：“ `0x8badf00d` ”，即“ `ate bad food` ”。

[苹果开发文档原文](https://developer.apple.com/library/archive/technotes/tn2151/_index.html) ：

> The exception code `0x8badf00d` indicates that an application has been terminated by iOS because a watchdog timeout occurred. The application took too long to launch, terminate, or respond to system events. One common cause of this is doing [synchronous networking on the main thread](http://developer.apple.com/library/ios/qa/qa1693/).
> Whatever operation is on `Thread 0` needs to be moved to a background thread, or processed differently, so that it does not block the main thread.

大致意思是说：如果我们的应用程序对一些特定的UI事件（比如启动、挂起、恢复、结束）响应不及时， `Watchdog` 会把我们的应用程序干掉，并生成一份响应的 `crash` 报告。

## 遇到的问题
 
应用 100% `Loss` 时完全无法启动，一直崩溃。彻底切断网络连接正常启动，调试模式状态下等待时间非常久，但可以启动，并伴随 `UI` 微卡。强烈的预感这是线程阻塞。

`注` ：用 `Xcode debug` 时 `watchdog` 并不运行，一定要把设备从 `Xcode` 断开来测试启动速度。

## 原因

首先看了 `crash log` ，就像猜测的那样，的确是卡在了主线程；意料之外的是，无数次闪退只留下了一份崩溃日志，如下所示：

![[3096256-e852d45b7bbfbb02.png]]

第一次见，读了一些资料大概才算是明白了这是怎么一回事。为了避免应用陷入错误状态导致界面无响应， `Apple` 设计了看门狗 ( `WatchDog` ) 机制。一旦超时，强制杀死进程。在不同的生命周期，触发看门狗机制的超时时间有所不同：
![[3096256-68c536289dbcc580.png]]


首先说一说异常编码，也是寓意颇深。 `8badf00d = ate bad food` ，大概是在说看门狗吃了坏的食物所以暴走了？！异常记录则表示这并不是一次崩溃（邪魅一笑：强制退出而已）。信息一栏指出时间限制为 `20 s` 。结合应用业务来看，表层原因在于：每次启动应用，首先进行一次模版同步，在此之前需要检测登录状况，通过 `RunLoop` 反复尝试直到收到响应为止。然而不幸的是，这一些都发生在主线程。

同步网络请求，主线程，超长超时时间，满足这三点，一定场景下几乎必然会触发看门狗机制。

## 对策

合理解决方案：

1. 异步网络请求：优点很多，最重要的是可以让你无忧无虑安全地访问网络，而无需担心线程。

2. 在非主线程中使用同步网络请求：如果异步运行你的网络代码比登天还难的话(也许你的应用是一个基于同步网络请求的大型移植项目)，退而求次，你也可以在次级线程中运行同步代码，也可以避免触发看门狗机制。

此外，一部分情况下，例如这次遇到登录和模版同步时触发看门狗，事实上，即使在运用到模版时再次请求也是勉强可行的，因此姑且先跳过网络请求也可以。此时，还以使用一种我认为是相对比较差的方案：

1. 通过 `RunLoop` 来操控一切，一旦超过既定的超时时间，就提示用户重试或者暂时先跳过网络请求。

应用的网络部分基于公司的通用框架，因此优先考虑在非主线程中进行网络请求来避免触发看门狗。
至于调试模式下为什么可以正常启动应用，完全是因为该模式下看门狗机制处于禁用状态。
此外，除了网络操作， `I/O` 读写文件和大规模运算等耗时任务也极有可能触发看门狗机制。合理处理线程，优化耗时任务，很大程度能避免不佳用户体验。

## Author

如果你有什么建议，可以关注我，直接留言，留言必回。

## 参考文章

[主线程上的同步网络请求](https://developer.apple.com/library/ios/qa/qa1693/_index.html)
[调试模式不发生崩溃](https://developer.apple.com/library/ios/qa/qa1592/_index.html)
[iOS的看门狗(watchdog)机制](https://www.jianshu.com/p/7f2ebc63c790)
[iOS的看门狗机制](https://ckitakishi.com/2016/08/17/WatchDog-%E6%9C%BA%E5%88%B6/)

[iOS watchdog (看门狗机制) - 简书](https://www.jianshu.com/p/6cf4aeced795)