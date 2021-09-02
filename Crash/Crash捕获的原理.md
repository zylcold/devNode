#crash 

要了解 Crash 捕获的原理，要先清楚几个基本概念，和它们之间的关系：

* 软件异常
	软件异常主要来源于两个 API 的调用 kill() 、 pthread_kill() , 而 iOS 中我们常常遇到的 NSException 未捕获、 abort()  函数调用等，都属于这种情况。比如我们常看到 Crash 堆栈中有 pthead_kill 方法的调用。

* 硬件异常
	硬件产生的信号始于处理器 trap，处理器 trap 是平台相关的。比如我们遇到的野指针崩溃大部分是硬件异常。

* Mach异常
	这里虽然叫异常，但是要和上面的两种分开来看。我们了解到苹果的内核 xnu 的核心是 Mach , 在 Mach 之上建立了 BSD 层。“Mach异常” 是 “Mach异常处理流程” 的简称。

* UNIX信号
	这就是我们常说的信号了，如 SIGBUS  SIGSEGV SIGABRT SIGKILL 等。
	
![[640 2.jpeg]]
**软件产生的信号** 的处理流程如下图， 就是用户态的 “UNIX信号” 处理流程

![[_640 2.jpeg]]


**硬件产生的信号** 的处理流程如下图：硬件错误被 Mach 层捕获，然后转换为对应的 “UNIX信号”。为了维护一个统一的机制，操作系统和用户产生的信号首先被转换为 "Mach异常"，然后再转换为信号。如下图所示:

![[__640 2.jpeg]]
上面两图来自 **“深入解析 Mac OS X & iOS 操作系统”** 一书。

由上图可以看到，无论是硬件产生的信号，还是软件产生的信号，都会走到 act_set_astbsd() 进而唤醒收到信号的进程的某一个线程。这个机制就给我们在“自身进程内捕获 Crash” 提供了可能性。就是可以通过拦截 “UNIX信号” 或 “Mach异常” 来捕获崩溃。

而且这里还有个小知识，当我们拦截信号处理之后是可以让程序不崩溃而继续运行的，但是不建议这样做，因为程序已经处于异常不可知状态。

PLCrashReporter  和 KSCrash 两个开源库都提供了 2 种方式拦截异常，包括 “Mach异常拦截” 和 “UNIX信号拦截”。说到这里有人可能困惑了，看上图里面最终都会转换为 “UNIX信号”， 是不是代表我们只用监听 “UNIX 信号” 就够了呢？ **为什么还要拦截 Mach 异常呢？**

有两个原因：

* 不是所有的 "Mach异常” 类型都映射到了 “UNIX信号”。 如 EXC_GUARD 。在苹果开源的 xnu 源码中可以看到这点。

* “UNIX信号” 在崩溃线程回调，如果遇到 Stackoverflow 问题，已经没有条件(栈空间)再执行回调代码了。

那么，是不是我们只拦截 “Mach异常” 就够了呢？也不是，用户态的软件异常是直接走信号流程的，如果不拦截信号可能导致这部分 Crash 丢失。

附 “Mach异常” 与 “UNIX信号” 的转换关系代码，来自 xnu 中的 bsd/uxkern/ux_exception.c ：

```c
switch(exception) {
case EXC_BAD_ACCESS:
    if (code == KERN_INVALID_ADDRESS)
        *ux_signal = SIGSEGV;
    else
        *ux_signal = SIGBUS;
    break;

case EXC_BAD_INSTRUCTION:
    *ux_signal = SIGILL;
    break;

case EXC_ARITHMETIC:
    *ux_signal = SIGFPE;
    break;

case EXC_EMULATION:
    *ux_signal = SIGEMT;
    break;

case EXC_SOFTWARE:
    switch (code) {

    case EXC_UNIX_BAD_SYSCALL:
    *ux_signal = SIGSYS;
    break;
    case EXC_UNIX_BAD_PIPE:
    *ux_signal = SIGPIPE;
    break;
    case EXC_UNIX_ABORT:
    *ux_signal = SIGABRT;
    break;
    case EXC_SOFT_SIGNAL:
    *ux_signal = SIGKILL;
    break;
    }
    break;

case EXC_BREAKPOINT:
    *ux_signal = SIGTRAP;
    break;
}
```

看到这里，有同学可能会说，还有 NSException 呢？我们都用 NSUncaughtExceptionHandler 来捕获异常 Crash 的。在前面就将 c++/ObjC 异常归类到了“软件异常” 类型，那是不是“捕获信号”就行了呢？为什么还要注册 NSUncaughtExceptionHandler 呢？是因为 CrashReporter 需要通过这个 handler 来获取异常相关信息和堆栈。

![[___640 2.jpeg]]