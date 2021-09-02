# 启动优化
![[8635a869aa06443c9ff91ecafb09d024~tplv-k3u1fbpfcp-zoom-1.image.png]]

既然要精细化分析，那么我们就需要标记出一小段时间，可以用 Point of interest 来标记。除此之外，System Trace 分析虚拟内存和线程状态都很管用：

* Virtual Memory： **主要关注 Page In** 这个事件，因为启动路径上有很多次 Page In，且相对耗时
* Thread State： **主要关注挂起和抢占两个状态，记住主线程不是一直在运行的**
* System Load 线程有优先级，高优先级的线程不应该超过系统核心数量
