#optimize #launch 
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