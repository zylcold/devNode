#webkit/issus  #crash  #workaround 

# 表现
在 iOS 8 系统上，completionHandler野指针崩溃

# 原因
另一个 crash 发生在 WKWebView 退出前调用：

```objc
-[WKWebView evaluateJavaScript: completionHandler:]
```

执行JS代码的情况下。WKWebView 退出并被释放后导致completionHandler变成野指针，而此时 javaScript Core 还在执行JS代码，待 javaScript Core 执行完毕后会调用completionHandler()，导致 crash。这个 crash 只发生在 iOS 8 系统上，参考Apple Open Source，在iOS9及以后系统苹果已经修复了这个bug，主要是对completionHandler block做了copy（refer: https://trac.webkit.org/changeset/179160 ）；

# 解决方案

对于iOS 8系统，可以通过在 completionHandler 里 retain WKWebView 防止 completionHandler 被过早释放。我们最后用 methodSwizzle hook 了这个系统方法：

```objc
+ (void) load {      
	[self jr_swizzleMethod:NSSelectorFromString(@"evaluateJavaScript:completionHandler:") withMethod:@selector(altEvaluateJavaScript:completionHandler:) error:nil]; 
}

/*
* fix: WKWebView crashes on deallocation if it has pending JavaScript evaluation
*/
- (void)altEvaluateJavaScript:(NSString *)javaScriptString completionHandler:(void(^)(id, NSError *))completionHandler {     
	id strongSelf = self;     
	[self altEvaluateJavaScript:javaScriptString completionHandler:^(id r, NSError *e) {         
		[strongSelf title];
		if(completionHandler) {             
			completionHandler(r, e);         
		}     
	}]; 
}
```