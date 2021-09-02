UIKit 初始化之后，就进入了我们熟悉的 UIApplicationDelegate 回调了，在这些会调里去做一些业务上的初始化：

* `willFinishLaunch`

* `didFinishLaunch`

* `didFinishLaunchNotification`

要特别提一下 `didFinishLaunchNotification` ，是因为大家在埋点的时候通常会忽略还有这个通知的存在，导致把这部分时间算到 UI 渲染里。