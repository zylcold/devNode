在我们自己研发 Crash 收集框架之前，最早肯定都会接入腾讯 Bugly、友盟等第三方日志框架来进行崩溃的收集和分析。
如果多个 Crash 收集框架存在时，往往会存在冲突。

不管是对于 Signal 捕获还是 NSException 捕获都会存在 handler 覆盖的问题，正确的做法应该是先判断是否有前者已经注册了 handler，如果有则应该把这个 handler 保存下来，在自己处理完自己的 handler 之后，再把这个 handler 抛出去，供前面的注册者处理。

```c
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

```

上面的是一个处理 Signal handler 冲突的大概代码思路，下面是 NSException handler 的处理思路，两者大同小异。

```c
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
```