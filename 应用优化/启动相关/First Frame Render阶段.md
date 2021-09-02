一般会用 Root Controller 的 viewDidApper 作为渲染的终点，但其实这时候首帧已经渲染完成一小段时间了，Apple 在 MetricsKit 里对启动终点定义是第一个 `CA::Transaction::commit()` 。

什么是 CATransaction 呢？我们先来看一下渲染的大致流程

iOS 的渲染是在一个单独的进程 RenderServer 做的，App 会把 Render Tree 编码打包给 RenderServer，RenderServer 再调用渲染框架(Metal/OpenGL ES)来生成 bitmap，放到帧缓冲区里，硬件根据时钟信号读取帧缓冲区内容，完成屏幕刷新。CATransaction 就是把一组 UI 上的修改，合并成一个事务，通过 commit 提交。

渲染可以分为四个步骤

* **Layout** （布局），源头是 Root Layer 调用 `[CALayer layoutSubLayers]` ，这时候 `UIViewController` 的 `viewDidLoad` 和 `LayoutSubViews` 会调用， `autolayout` 也是在这一步生效
* **Display** （绘制），源头是 Root Layer 调用 `[CALayer display]` ，如果 View 实现了 `drawRect` 方法，会在这个阶段调用
* **Prepare** （准备），这个过程中会完成图片的解码
* **Commit** （提交），打包 Render Tree 通过 XPC 的方式发给 Render Server