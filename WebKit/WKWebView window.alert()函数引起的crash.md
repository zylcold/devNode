#webkit/issus  #workaround #crash 

# 表现
* JS调用window.alert()函数引起

WKWebView 放量后，外网新增了一些 crash, 其中一类 crash 的主要堆栈如下：

```js
...28UIKit0x0000000190513360UIApplicationMain +208

29Qzone0x0000000101380570main (main.m:181)30libdyld.dylib0x00000001895205b8_dyld_process_info_notify_release +36

Completion handler passed to -[QZWebController webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:] was not called
```


# 原因

主要是JS调用window.alert()函数引起的，从 crash 堆栈可以看出是 WKWebView 回调函数:

```objc
+ (void) presentAlertOnController:(nonnull UIViewController*)parentController title:(nullable NSString*)title message:(nullable NSString *)message handler:(nonnullvoid(^)())completionHandler;
```

completionHandler 没有被调用导致的。在适配 WKWebView 的时候，我们需要自己实现该回调函数，window.alert()才能调起 alert 框，我们最初的实现是这样的：

```objc
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void(^)(void))completionHandler {    
	UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"" message:message  preferredStyle:UIAlertControllerStyleAlert];     
	[alertController addAction:[UIAlertAction actionWithTitle:@"确认" style:UIAlertActionStyleCancel handler:^(UIAlertAction *action) { completionHandler(); }]];     
	[self presentViewController:alertController animated:YES completion:^{}]; 
}
```

如果 WKWebView 退出的时候，JS刚好执行了window.alert(), alert 框可能弹不出来，completionHandler 最后没有被执行，导致 crash；另一种情况是在 WKWebView 一打开，JS就执行window.alert()，这个时候由于 WKWebView 所在的 UIViewController 出现（push或present）的动画尚未结束，alert 框可能弹不出来，completionHandler 最后没有被执行，导致 crash。

# 解决方案

我们最终的实现大致是这样的：

```objc
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void(^)(void))completionHandler {
	if(/*UIViewController of WKWebView has finish push or present animation*/){         
		completionHandler();
		return;     
	}     
	UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"" message:message preferredStyle:UIAlertControllerStyleAlert];     
	[alertController addAction:[UIAlertAction actionWithTitle:@"确认"style:UIAlertActionStyleCancel handler:^(UIAlertAction *action) { completionHandler(); }]];
	if(/*UIViewController of WKWebView is visible*/) {
		[self presentViewController:alertController animated:YES completion:^{}];
	}else {
		completionHandler()
	}; 
}
```

确保上面两种情况下 completionHandler 都能被执行，消除了 WKWebView 下弹 alert 框的 crash，WKWebView 下弹 confirm 框的 crash 的原因与解决方式与 alert 类似。



