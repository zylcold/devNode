#crash

## 分类
Crash分为两种，未捕获的[[Objective-C异常]]和[[Mach异常]]

## Crash分析步骤
1. 保存并收集[[符号表dSYM]]
2. [[Crash捕获的原理]]，收集[[Crash日志]]
3. 对[[Crash日志符号化]]

## 常见的Objective-C异常
[[NSInvalidArgumentException]]
[[NSRangeException]]
[[NSGenericException]]
[[NSInternalInconsistencyException]]
[[NSFileHandleOperationException]]
[[NSInvalidArgumentException]]
[[NSMallocException]]


## 常见的Mach异常
[[SIGHUP]]
[[SIGINT]]
[[SIGQUIT]]
[[SIGABRT]]
[[SIGBUS]]
[[SIGFPE]]
[[SIGKILL]]
[[SIGSEGV]]
[[SIGPIPE]]


## 常见的Crash案例
[[iOS13 一次Crash定位 - 被释放的NSURL.host]]
[[反编译分析Xcode8的Bug, release下连续两次调用有二级指针参数的空方法会Crash]]
[[了解和分析iOS Crash Report - 掘金]]
[[iOS App 后台任务的坑]]
[[iOS watchdog (看门狗机制) - 简书]]
[[你的 App 在 iOS 13 上被卡死了吗？]]
[[谈nonatomic非线程安全问题]]
[[Dealloc 时取 weak self 引起崩溃]]
[[iOS 野指针处理]]
[[调试内存不足问题：使用运行时魔法捕获布局反馈循环]]

# 如何处理OOM
[[如何判定发生了OOM]]
[[OOM与内存]]