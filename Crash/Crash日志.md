#crash #log
# [[收集Crash日志]]

# 日志内容Demo

日志主要分为六个部分： **进程信息**、**基本信息**、**异常信息**、**线程回溯**、**线程状态** 和 **二进制映像** 。下面是从某APP具体的Crash日志抽出的主要信息，展示如下：
### 1、进程信息
```
Hardware Model: iPhone9,2
Process: AppName [3580]
Path: /var/containers/Bundle/Application/C7B90C8A-E269-4413-A011-552971D1ED39/AppName.app
Identifier: xxxx.xxx.xxxx.xxx
Version: xx.xx
Code Type: ARM-64 (Native)
Parent Process:  [1]
```

## 2、基本信息
```
Date/Time: 2017-05-22 03:05:06.743 +0800
OS Version: iPhone OS 10.2.1 (14D27)
```

### 3、异常信息
```
Exception Type: NSInvalidArgumentException(SIGABRT)
Exception Codes: -[NSNull integerValue]: unrecognized selector sent to instance 0x1a9d88ef8 at 0x00000001835c7014
Crashed Thread: 0
```

### 4、线程回溯 （展示发生Crash线程的回溯信息，其他略）
```
Thread 0 Crashed: 
0  libsystem_kernel.dylib         0x00000001835c7014 __pthread_kill + 4
1  libsystem_c.dylib              0x000000018353b400 abort + 140
2  AppName                         0x0000000100a26704 0x0000000100028000 + 10479360
3  CoreFoundation                 0x00000001845f9538 ___handleUncaughtException +  644
2  CoreFoundation                 0x0000000184600268 ___methodDescriptionForSelector
3  CoreFoundation                 0x00000001845fd270 ____forwarding___ +  916
4  CoreFoundation                 0x00000001844f680c _CF_forwarding_prep_0 + 80
5  AppName                         0x0000000100205280 0x0000000100028000 + 1954432
6  AppName                         0x00000001002ae59c 0x0000000100028000 + 2647440
7  AppName                         0x0000000100482944 0x0000000100028000 + 4565312
16 CoreFoundation                 0x00000001845a6810 ___CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__ +  12
     +  12
17 CoreFoundation                 0x00000001845a43fc ___CFRunLoopRun +  1660
18 CoreFoundation                 0x00000001844d22b8 CFRunLoopRunSpecific + 436
```

### 5、进程状态（展示部分）
```
Thread 0 crashed with ARM 64 Thread State:
     x0:  000000000000000000    x1: 000000000000000000    x2: 000000000000000000     x3: 0xffffffffffffffff
     x4:  0x0000000000000010    x5: 0x0000000000000020    x6: 000000000000000000     x7: 000000000000000000
     x8:  0x0000000008000000    x9: 0x0000000004000000   x10: 000000000000000000    x11: 0x00000001ac336c83
    x12: 0x00000001ac336c83    x13: 0x0000000000000018   x14: 0x0000000000000001    x15: 0x0000000000000881
    x16: 0x0000000000000148    x17: 000000000000000000   x18: 000000000000000000    x19: 0x0000000000000006
```

### 6、二进制映像 （展示部分）
```
Binary Images:
0x100028000 - 0x1011dbfff +AppName arm64 <ff7a4009322b386ea3e8e9a3fde05be4> /var/containers/Bundle/Application/C7B90C8A-E269-4413-A011-552971D1ED39/AppName.app/AppName
0x18368a000 - 0x183693fff  libsystem_pthread.dylib arm64 <258dc0c51499393bba7ba3e83dc5bfbb> /usr/lib/system/libsystem_pthread.dylib
0x1835a8000 - 0x1835ccfff  libsystem_kernel.dylib arm64 <1baa3f5629c43467879d4cf463a20b06> /usr/lib/system/libsystem_kernel.dylib
0x1834b1000 - 0x1834b5fff  libdyld.dylib arm64 <db54f120486a3710a684ce8bb1cb9d71> /usr/lib/system/libdyld.dylib
0x1834d8000 - 0x183556fff  libsystem_c.dylib arm64 <8a5a190d70563f3c8d4ce16cab74f599> /usr/lib/system/libsystem_c.dylib
0x183481000 - 0x1834b0fff  libdispatch.dylib arm64 <fb1d0baf642337d1bea0af309586df97> /usr/lib/system/libdispatch.dylib
0x183028000 - 0x183401fff  libobjc.A.dylib arm64 <538f809dcd7c35ceb59d99802248f045> /usr/lib/libobjc.A.dylib
```

# 日志内容组成分析

整个日志内容中，直接和Crash信息相关，最能帮助开发者定位问题部分是： **异常信息** 和 **线程回溯** 部分的内容。

1) 进程信息：发生Crash闪退进程的相关信息

* **Hardware Model** : 标识设备类型。 如果很多崩溃日志都是来自相同的设备类型，说明应用只在某特定类型的设备上有问题。上面的日志里，崩溃日志产生的设备是iPhone 7 Plus (iPhone 7 Plus 也是2个版本 iPhone9,2 和 iPhone9,4. 硬件代号为 D11AP 和 D111AP. 型号有: A1661, A1784, A1785 和 A1786. )

* **Process** 是应用名称。中括号里面的数字是闪退时应用的进程ID。

2) 基本信息：给出了一些基本信息，包括闪退发生的日期和时间，设备的iOS版本。

3) 异常信息：闪退发生时抛出的异常类型。还能看到异常编码和抛出异常的线程。

```
//以上面内容中的异常信息为例：
Exception Type: NSInvalidArgumentException(SIGABRT)
Exception Codes: -[NSNull integerValue]: unrecognized selector sent to instance 0x1a9d88ef8 at 0x00000001835c7014
Crashed Thread: 0
```

* **Exception Type异常类型：** 通常包含1.7中的Signal信号和EXC_BAD_ACCESS，NSRangeException等。
* **Exception Codes** ：异常编码：
* **Crashed Thread** ：发生Crash的线程id

4) 线程回溯：回溯是闪退发生时所有活动帧清单。它包含闪退发生时调用函数的清单。

5) 线程状态：闪退时寄存器中的值。一般不需要这部分的信息，因为回溯部分的信息已经足够让你找出问题所在。

6) 二进制映像：闪退时已经加载的二进制文件。

# 异常信息解读

##  1、Exception Type（异常类型）

* **Exception Type** ：通常包含Signal信号 和 EXC_BAD_ACCESS，NSRangeException等。
异常类型 可能的原因 调试方法     EXC_CRASH unrecognized selector All Exception Point   EXC_BAD_ACCESS 内存访问错误 NSZombie   SIGSEGV 引用了released对象 / 引用未init的对象 / 数组越界/ 试图往没有写权限的内存地址写数据 NSZombie   SIGABRT 逻辑错误导致的Crash,比如尝试多次释放同一个没存 逻辑检查   SIGPIPE TCP突然断开，再发送数据 添加signal（SIGPIPE，XX）

具体信号说明参见 [iOS异常捕获](https://link.juejin.im?target=http%3A%2F%2Fwww.iosxxx.com%2Fblog%2F2015-08-29-iosyi-chang-bu-huo.html)

## 2、Exception Code（异常编码）

* **Exception Code** ：以一些文字开头，紧接着是一个或多个十六进制值。这些数值说明了Crash发生的本质。

* 从Exception Code中，可以区分出Crash是因为程序错误、非法内存访问还是其他原因。常见的异常编码如下表：

异常编码 描述     0x8badf00d ate bad food ，表示应用是因为发生watchdog超时而被iOS终止的。通常是应用花费太多时间而无法启动、终止或响应用系统事件。   0xdeadfa11 dead fall，用户强制退出。   0xbaaaaaad 用户按住Home键和音量键，获取当前内存状态，不代表崩溃。   0xbad22222 VoIP 应用因为过于频繁重启而被终止   0xc00010ff cool off，因为太烫了被干掉   0xdead10cc dead lock，表明应用因为在后台运行时占用系统资源（如通讯录数据库）   0xbbadbeef bad beef，发生致命错误

**说明1** ：详细的异常编码代表的含义请参考： [Hexspeak](https://link.juejin.im?target=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FHexspeak)

**说明2** ：在后台任务列表中关闭已挂起的应用不会产生崩溃日志。 因为应用一旦被挂起，它何时被终止都是合理的。所以不会产生崩溃日志。

## 四、Crash日志符号化

##### 1、概述

线程回溯部分内容如下：

```
5  AppName                         0x0000000100205280 0x0000000100028000 + 1954432
6  AppName                         0x00000001002ae59c 0x0000000100028000 + 2647440
```

**这两条记录包括四列**:（以第一条记录为例子）

* 帧编号—— 5（数字越小，发生时间越晚，发生顺序越往后，越好锁定问题的范围）
* 二进制库的名称 ——此处是 AppName.
* 调用方法的地址 ——此处是 0x0000000100205280.
* 第四列分为两个子列，一个基本地址和一个偏移量。此处是 x0000000100028000 + 1954432, 第一个数字指向文件，第二个数字指向文件中的代码行。

**说明1** ：线程回溯部分并不是我们习惯使用 **方法名和行数** ，而是 **十六进制地址** 。所以我们在分析Crash前需要将 **这些十六进制地址转化成方法名称和行数** ，改过程被称为 **符号化** 。

**说明2** ：符号化Crash日志需要获取对应的 **应用二进制文件** 以及 **生成二进制文件时产生的 .dSYM 文件（符号表）** 。必需完全匹配才行。否则，日志将无法被完全符号化。

**说明3** ： Xcode编译项目后，会得到同名的 dSYM 文件（符号表），dSYM 文件（符号表）是保存 16 进制函数地址映射信息的中转文件，我们调试的 symbols 都会包含在这个文件中，并且每次编译项目的时候都会生成一个新的 dSYM 文件，位于 /Users/<用户名>/Library/Developer/Xcode/Archives 目录下，对于每一个发布版本我们都很有必要保存对应的 Archives 文件。

**说明4** ：符号化可以使用Xcode的两种命令 **symbolicatecrash命令** + **atos命令** 