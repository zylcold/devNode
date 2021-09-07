#crash 

Crash 日志从哪来？一般有 2 个渠道:

* **苹果收集的 Crash 日志**

	* 用户手机上 设置 -> 隐私 -> 分析 里面的，可以连接电脑 Xcode 导出。
	* 在 Xcode -> Window -> Organizer -> Crashes 里面可以查看

* **自己应用内收集的**

	* 接入一些 APM 产品， 如 EMAS、mPaaS、phabricator 等。
	* 接入 PLCrashReporter 、 KSCrash 等 SDK 进行收集，上报到自建平台统计

两者各有利弊，但是二者的[[Crash捕获的原理|捕获原理]]是差不多的。

* **苹果的日志**
	* 优点: 理论上捕获类型最全，因为是 launchd 进程捕获的日志。
	* 缺点：不是全量日志，因为需要用户隐私授权才会上报，没有数据化支撑。

* **自己收集的**
	* 优点：可以自建数据化支撑，获取 Crash 率等指标。
	* 缺点：存在无法捕获的 Crash 的类型。
