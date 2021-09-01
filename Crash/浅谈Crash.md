#crash

## 分类
Crash分为两种，未捕获的[[Objective-C异常]]和[[Mach异常]]

## Crash日志收集

1. [[苹果Crash收集服务]]

2. 自行实现Crash收集及上报框架
	实现原理上面已详细描述，适合人手充足，技术储备足够的团队使用，推荐指数五颗星。

3. 第三方crash收集服务
	腾讯bugly、友盟等Crash收集服务比较完善，作为开发者省心省力，适合个人或者对隐私性要求不高的团队使用，推荐指数五颗星。


Crash日志收集的冲突

以下内容来自网络文章

在我们自己研发 Crash 收集框架之前，最早肯定都会接入腾讯 Bugly、友盟等第三方日志框架来进行崩溃的收集和分析。如果多个 Crash 收集框架存在时，往往会存在冲突。
不管是对于 Signal 捕获还是 NSException 捕获都会存在 handler 覆盖的问题，正确的做法应该是先判断是否有前者已经注册了 handler，如果有则应该把这个 handler 保存下来，在自己处理完自己的 handler 之后，再把这个 handler 抛出去，供前面的注册者处理。

typedef void (*SignalHandler)(int signo, siginfo_t *info, void *context);

static SignalHandler previousSignalHandler = NULL;

+ (void)installSignalHandler {
struct sigaction old_action;
sigaction(SIGABRT, NULL, &old_action);
if (old_action.sa_flags & SA_SIGINFO) {
previousSignalHandler = old_action.sa_sigaction;
}

LDAPMSignalRegister(SIGABRT);
// .......

}
static void LDAPMSignalRegister(int signal) {
struct sigaction action;
action.sa_sigaction = LDAPMSignalHandler;
action.sa_flags = SA_NODEFER | SA_SIGINFO;
sigemptyset(&action.sa_mask);
sigaction(signal, &action, 0);
}
static void LDAPMSignalHandler(int signal, siginfo_t* info, void* context) {
// 获取堆栈，收集堆栈
........

LDAPMClearSignalRigister();

// 处理前者注册的 handler
if (previousSignalHandler) {
previousSignalHandler(signal, info, context);
}
}

上面的是一个处理 Signal handler 冲突的大概代码思路，下面是 NSException handler 的处理思路，两者大同小异。

static NSUncaughtExceptionHandler *previousUncaughtExceptionHandler;

static void LDAPMUncaughtExceptionHandler(NSException *exception) {
// 获取堆栈，收集堆栈
// ......
// 处理前者注册的 handler
if (previousUncaughtExceptionHandler) {
previousUncaughtExceptionHandler(exception);
}
}

+ (void)installExceptionHandler {
previousUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
NSSetUncaughtExceptionHandler(&LDAPMUncaughtExceptionHandler);
}

堆栈符号解析

分为两种情景讨论：

1、Crash收集上报的堆栈信息解析

这种信息一般还原度比较高，基本上都能给出崩溃的具体定位信息，一些需要解析堆栈符号的解决方法如下：
以下内容来自文章《iOS异常捕获-堆栈信息的解析》

异常信息有三种类型：
1.已标记错误位置的:

test 0x000000010bfddd8c -[ViewController viewDidLoad] + 8588

- 这种信息已经很明确了，不用解析

2.有模块地址的情况：

test 0x00000001018157dc 0x100064000 + 24844252

以上面为例子，从左到右依次是：
二进制库名（test），调用方法的地址（0x00000001018157dc），模块地址（0x100064000）+偏移地址（24844252）

3.无模块地址的情况：

test 0x00000001018157dc test + 24844252

解析堆栈信息
dSYM符号表获取，xcode->window->organizer->右键你的应用 show finder->右键.xcarchive 显示包内容->dSYMs->test.app.dYSM

然后使用atos命令来符号化某个特定模块加载地址

atos [-arch 架构名] [-o 符号表] [-l 模块地址] [方法地址]

使用终端，进到test.app.dYSM所在目录

一.如果是有模块地址的情况，运行：

atos -arch arm64 -o test.app.dSYM/Contents/Resources/DWARF/test -l 0x100064000 0x00000001018157dc

二.如果是无模块地址的情况

1.先将偏移地址转为16进制：

24844252 = 0x17B17DC

2.然后用方法的地址-偏移地址，得到的就是模块地址

0x00000001018157dc - 0x17B17DC = 0x100064000

3.最后运行：

atos -arch arm64 -o test.app.dSYM/Contents/Resources/DWARF/test -l 0x100064000 0x00000001018157dc

2、苹果自带Crash日志上报的堆栈信息解析

以下内容来自文章《iOS Crash 捕获及堆栈符号化思路剖析》

有四种常见的方法：
* symbolicatecrash
* mac 下的 atos 工具
* linux 下的 atos 的替代品 atosl
* 通过 dSYM 文件提取地址和符号的对应关系，进行符号还原

以上方案都有对应的应用场景，对于线上的 Crash 堆栈符号还原，主要采用的还是后三种方案。atos 和 atosl 的使用方法很类似，以下是 atos 的一个示例。

atos -o MonitorExample 0x0000000100062ac4 ARM-64 -l 0x100058000

// 还原结果
-[GYRootViewController tableView:cellForRowAtIndexPath:] (in GYMonitorExample) (GYRootViewController.m:41)

但是 atos 是Mac上一个工具，需要使用 Mac 或者黑苹果来进行解析工作，如果由后台来做解析工作，往往需要一套基于 Linux 的解析方案，这个时候可以选择 atosl，但是这个库已经有多年没有更新了，同时基于我司的尝试， atosl 好像不太支持 arm64 架构，所以我们放弃了该方案。
最终使用了第四个方案，提取 dSYM 的符号表，可以自己研发工具，也可以直接使用 bugly 和友盟的工具，下面是提取出来的符号表。第一列是起始内存地址，第二列是结束地址，第三列是对应的函数名、文件名以及行号。

a840 a854 -[GYRootViewController tableView:cellForRowAtIndexPath:] GYRootViewController.m:41
a854 a858 -[GYRootViewController tableView:cellForRowAtIndexPath:] GYRootViewController.m:42
a858 a87c -[GYRootViewController tableView:cellForRowAtIndexPath:] GYRootViewController.m:42
a87c a894 -[GYRootViewController tableView:cellForRowAtIndexPath:] GYRootViewController.m:42
a894 a8a0 -[GYRootViewController tableView:cellForRowAtIndexPath:] GYRootViewController.m:42
aa3c aa80 -[GYFilePreviewViewController initWithFilePath:] GYRootViewController.m:21
aa80 aaa8 -[GYFilePreviewViewController initWithFilePath:] GYFilePreviewViewController.m:23
aaa8 aab8 -[GYFilePreviewViewController initWithFilePath:] GYFilePreviewViewController.m:23
aab8 aabc -[GYFilePreviewViewController initWithFilePath:] GYFilePreviewViewController.m:24
aabc aac8 -[GYFilePreviewViewController initWithFilePath:] GYFilePreviewViewController.m:24

因为程序每次启动基地址都会变化，所以上面提到的地址是相对偏移地址，在我们获取到崩溃堆栈地址后，可以根据堆栈中的偏移地址来与符号表中的地址来做匹配，进而找到堆栈所对应的函数符号。比如下面的第四行，偏移为 43072 转换为十六进制就是 a840，用 a840 去上面的符号表中找对应关系，会发现对应着 -[GYRootViewController tableView:cellForRowAtIndexPath:]，基于这种方式，就可以将堆栈地址完全还原为函数符号啦。

0 libsystem_kernel.dylib 0x0000000186cfd314 0x186cde000 + 127764
1 Foundation 0x00000001887f5590 0x1886ec000 + 1086864
2 GYMonitorExample 0x00000001000da4ac 0x1000d0000 + 42156
3 GYMonitorExample 0x00000001000da840 0x1000d0000 + 43072

野指针Crash的解决方案

野指针是最玄的Crash，表现在飘忽不定（往往不是100%复现）、难以定位（往往崩溃时的堆栈信息并不能定位到具体的代码行）、难以消除（往往负责的业务和代码逻辑使其隐藏得很深，在测试的覆盖范围外出现）。

既然不能根治，那就尽量把它抑制吧，下面是一些优秀的文章：

如何定位Obj-C野指针随机Crash(一)：先提高野指针Crash率
如何定位Obj-C野指针随机Crash(二)：让非必现Crash变成必现
如何定位Obj-C野指针随机Crash(三)：加点黑科技让Crash自报家门