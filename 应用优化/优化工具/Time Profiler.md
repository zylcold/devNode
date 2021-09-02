# 启动优化检测

Time Profiler 是大家日常性能分析中用的比较多的工具，通常会选择一个时间段，然后聚合分析调用栈的耗时。但 **Time Profiler 其实只适合粗粒度的分析** ，为什么这么说呢？我们来看下它的实现原理：

**默认 Time Profiler 会 1ms 采样一次，只采集在运行线程的调用栈，最后以统计学的方式汇总** 。比如下图中的 5 次采样中，method3 都没有采样到，所以最后聚合到的栈里就看不到 method3。所以 Time Profiler 中的看到的时间，并不是代码实际执行的时间，而是栈在采样统计中出现的时间。

![[2675a2c20cda42c1bbd9e8afb9872c48~tplv-k3u1fbpfcp-zoom-1.image.png]]

Time Profiler 支持一些额外的配置， **如果统计出来的时间和实际的时间相差比较多，可以尝试开启** ：

* High Frequency，降低采样的时间间隔
* Record Kernel Callstacks，记录内核的调用栈
* Record Waiting Thread，记录被 block 的线程