
## 简介
在OC层面(iOS库、第三方库出现错误抛出)的异常称为OC异常。

比如：
```objc
NSArray * array= @[@“s",@“x",@“m"];
						 
[array objectAtIndex:4];
```

OC异常可以用try-catch抓住：

```objc
@try {
	
	NSArray * array= @[@“s",@“x",@“m"];
	[array objectAtIndex:4];
							 
} @catch (NSException *exception) {
	
	NSLog(@"%@",exception);
	
}
```

使用此方法可以抓到当前抛出的异常并阻止程序崩溃，然而苹果并不推荐这样去做。

```
-[__NSArrayI objectAtIndex:]: index 4 beyond bounds [0 .. 2]
```


## 常见的OC异常
[[NSInvalidArgumentException]]
[[NSRangeException]]
[[NSGenericException]]
[[NSInternalInconsistencyException]]
[[NSFileHandleOperationException]]
[[NSInvalidArgumentException]]
[[NSMallocException]]


## OC异常的抓取和分析
在debug环境下，OC异常导致崩溃时Xcode控制台会输出完整的异常信息，比如：
```
Terminating app due to uncaught exception ‘NSRangeException’, reason: ‘this is reason description’，
```
包括Exception的类型、原因和发生异常的完整堆栈。
这些信息一般来说都足够详细，足够我们轻易地找到异常的位置并进行修复。


非debug环境下，可以通过注册 NSUncaughtExceptionHandler 捕获异常信息。虽然无法阻止APP崩溃，但是可以获取异常信息并进行收集，下次启动APP时进行上报，方便开发者进行错误跟踪及修复，这就是常用Crash收集工具所做的事情。

```objc

void InstallUncaughtExceptionHandler(void) {
	NSSetUncaughtExceptionHandler(&handleUncaughtException);
}

void handleUncaughtException(NSException *exception) {
	NSString * crashInfo = [NSString stringWithFormat:@"Exception name：%@\nException reason：%@\nException stack：%@",[exception name], [exception reason], [exception callStackSymbols]];
	[WZCrashReporter saveCrash:crashInfo];
}

```
