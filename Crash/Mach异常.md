Mach异常是指最底层的内核级异常。


# Mach异常 和 Unix信号

iOS Crash发生时，先产生 **Mach异常** (最底层的内核级异常)，然后Mach异常在host层被ux_exception转换为相应的 **Unix信号** ，并通过threadsignal将信号投递到出错的线程。

在捕获Crash事件时，优选Mach异常。因为 **Mach异常处理会先于Unix信号处理发生** ，如果Mach异常的handler让程序exit了，那么Unix信号就永远不会到达这个进程了。而转换Unix信号是为了兼容更为流行的POSIX标准(SUS规范)，这样就不必了解Mach内核也可以通过Unix信号的方式来兼容开发。

在方案实现时，通过捕获 **Mach异常+Unix信号组合方式** 来捕获Crash事件。在选择具体方案时，可以选择 [PLCrashReporter](https://link.juejin.im?target=https%3A%2F%2Fwww.plcrashreporter.org%2F) 这样优秀的开源项目，也可以选择 [友盟](https://link.juejin.im?target=http%3A%2F%2Fwww.umeng.com%2F)、[Bugly](https://link.juejin.im?target=http%3A%2F%2Fbugly.qq.com%2F) 这类完善的Crash上报和统计的产品(试项目需求而定)。


最常见的Mach异常：

EXC_BAD_ACCESS (Bad Memory Access)
这种内存访问异常分为访问非法地址(SIGBUS信号)和访问了被回收掉的内存(SIGSEGV信号)，实际开发中遇到的错误通常令人莫名其妙，往往需要大量时间来排查，非常头疼。
EXC_BAD_ACCESS后面通常带有code来帮助我们判断到底是什么错误，比如EXC_I386_GPFLT指访问了一块已经不属于你的内存。

EXC_BAD_INSTRUCTION
运行了非法的指令，往往是运行指令的参数不对（0或者nil的参数）

EXC_RESOURCE
程序资源上限（cpu占用过高或者内存不足）
EXC_GUARD

一些C函数访问错误导致的异常。

0x00000020奇怪异常集合，常见的是由于主线程阻塞看门狗杀死了APP《 [Exception Type: 00000020：什么是看门狗机制](https://link.jianshu.com/?t=https://www.cnblogs.com/qingche/p/5209226.html) 》


Unix Signal Exception

从Mach异常最终会转化成Unix信号投递到出错的线程

1. OC异常并不是真正的异常，但是当一个OC异常被抛出到最外层还没被捕获，程序会强行发送SIGABRT信号中断程序。
2. Mach异常没有比较便利的捕获方式，既然它最终会转化成信号，我们也可以通过捕获信号，来捕获 Crash 事件。

## 捕获
iOS提供了signal方法来注册一个处理函数，在处理函数中，使用execinfo中的 backtrace_symbols取出汇编层程序的堆栈信息。

代码如下：
```c
void InstallSignalHandler(void) {
	signal(SIGHUP, handleSignalException);
	signal(SIGINT, handleSignalException);
	signal(SIGQUIT, handleSignalException);

	signal(SIGABRT, handleSignalException);
	signal(SIGILL, handleSignalException);
	signal(SIGSEGV, handleSignalException);
	signal(SIGFPE, handleSignalException);
	signal(SIGBUS, handleSignalException);
	signal(SIGPIPE, handleSignalException);
}

void handleSignalException(int signal) {
    NSMutableString * crashInfo = [[NSMutableString alloc]init];
    [crashInfo appendString:[NSString stringWithFormat:@"signal:%d\n",signal]];
    [crashInfo appendString:@"Stack:\n"];
    void* callstack[128];
    int i, frames = backtrace(callstack, 128);
    char** strs = backtrace_symbols(callstack, frames);
    for (i = 0; i <frames; ++i) {
        [crashInfo appendFormat:@"%s\n", strs[i]];
    }
    [WZCrashReporter saveCrash:crashInfo];
}

```

常用信号代表的含义：
[[SIGHUP]]
[[SIGINT]]
[[SIGQUIT]]
[[SIGABRT]]
[[SIGBUS]]
[[SIGFPE]]
[[SIGKILL]]
[[SIGSEGV]]
[[SIGPIPE]]
