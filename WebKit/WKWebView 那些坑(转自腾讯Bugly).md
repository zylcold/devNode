#webview #workaround

### 导语
WKWebView 是苹果在 WWDC 2014 上推出的新一代 webView 组件，用以替代 UIKit 中笨重难用、内存泄漏的 UIWebView。WKWebView拥有60fps滚动刷新率、和 safari 相同的 JavaScript 引擎等优势。

简单的适配方法本文不再赘述，主要来说说适配 WKWebView 过程中填过的坑以及善待解决的技术难题。

### 1、WKWebView 白屏问题

WKWebView 自诩拥有更快的加载速度，更低的内存占用，但实际上 WKWebView 是一个多进程组件，Network Loading 以及 UI Rendering 在其它进程中执行。初次适配 WKWebView 的时候，我们也惊讶于打开 WKWebView 后，App 进程内存消耗反而大幅下降，但是仔细观察会发现，Other Process 的内存占用会增加。在一些用 webGL 渲染的复杂页面，使用 WKWebView 总体的内存占用（App Process Memory + Other Process Memory）不见得比 UIWebView 少很多。

在 UIWebView 上当内存占用太大的时候，App Process 会 crash；而在 WKWebView 上当总体的内存占用比较大的时候，WebContent Process 会 crash，从而出现白屏现象。在 WKWebView 中加载下面的测试链接可以稳定重现白屏现象:

[http://people.mozilla.org/~rnewman/fennec/mem.html](https://www.jianshu.com/p/bb20ff351fa2)

这个时候 WKWebView.URL 会变为 nil, 简单的 reload 刷新操作已经失效，对于一些长驻的H5页面影响比较大。

**我们最后的解决方案是：**

A、借助 WKNavigtionDelegate

iOS 9以后 WKNavigtionDelegate 新增了一个回调函数：

```objc
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView API_AVAILABLE(macosx(10.11),ios(9.0));
```

当 WKWebView 总体内存占用过大，页面即将白屏的时候，系统会调用上面的回调函数，我们在该函数里执行`[webView reload]`(这个时候 webView.URL 取值尚不为 nil）解决白屏问题。在一些高内存消耗的页面可能会频繁刷新当前页面，H5侧也要做相应的适配操作。

B、检测 webView.title 是否为空

并不是所有H5页面白屏的时候都会调用上面的回调函数，比如，最近遇到在一个高内存消耗的H5页面上 present 系统相机，拍照完毕后返回原来页面的时候出现白屏现象（拍照过程消耗了大量内存，导致内存紧张，WebContent Process 被系统挂起），但上面的回调函数并没有被调用。在WKWebView白屏的时候，另一种现象是 webView.titile 会被置空, 因此，可以在 viewWillAppear 的时候检测 webView.title 是否为空来 reload 页面。

综合以上两种方法可以解决绝大多数的白屏问题。

### 2、WKWebView Cookie 问题

Cookie 问题是目前 WKWebView 的一大短板

2.1、WKWebView Cookie存储

业界普遍认为 WKWebView 拥有自己的私有存储，不会将 Cookie 存入到标准的 Cookie 容器 **NSHTTPCookieStorage** 中。

实践发现 WKWebView 实例其实也会将 Cookie 存储于 NSHTTPCookieStorage 中，但存储时机有延迟，在iOS 8上，当页面跳转的时候，当前页面的 Cookie 会写入 NSHTTPCookieStorage 中，而在 iOS 10 上，JS 执行 document.cookie 或服务器 set-cookie 注入的 Cookie 会很快同步到 NSHTTPCookieStorage 中，FireFox 工程师曾建议通过 reset WKProcessPool 来触发 Cookie 同步到 NSHTTPCookieStorage 中，实践发现不起作用，并可能会引发当前页面 session cookie 丢失等问题。

**WKWebView Cookie 问题在于 WKWebView 发起的请求不会自动带上存储于 NSHTTPCookieStorage 容器中的 Cookie** 。

比如，NSHTTPCookieStorage 中存储了一个 Cookie:

name=Nicholas;value=test;domain=y.qq.com;expires=Sat,02May201923:38:25GMT；

通过 UIWebView 发起请求 [http://y.qq.com，](https://www.jianshu.com/p/bb20ff351fa2) 则请求头会自动带上 cookie: Nicholas=test；

而通过 WKWebView发起请求 [http://y.qq.com，](https://www.jianshu.com/p/bb20ff351fa2) 请求头不会自动带上 cookie: Nicholas=test。

2.2、WKProcessPool

苹果开发者文档对 WKProcessPool 的定义是： [A WKProcessPool object represents a pool of Web Content process](https://www.jianshu.com/p/bb20ff351fa2). 通过让所有 WKWebView 共享同一个 WKProcessPool 实例，可以 **实现多个 WKWebView 之间共享 Cookie（session Cookie and persistent Cookie）数据** 。不过 WKWebView WKProcessPool 实例在 app 杀进程重启后会被重置，导致 WKProcessPool 中的 Cookie、session Cookie 数据丢失，目前也无法实现 WKProcessPool 实例本地化保存。

2.3、Workaround

由于许多 H5 业务都依赖于 Cookie 作登录态校验，而 WKWebView 上请求不会自动携带 Cookie, 目前的主要解决方案是：

a、WKWebView loadRequest 前，在 request header 中设置 Cookie, 解决首个请求 Cookie 带不上的问题；

```objc
WKWebView *webView = [WKWebView new];

NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://h5.qzone.qq.com/mqzone/index"]];

[request addValue:@"skey=skeyValue" forHTTPHeaderField:@"Cookie"];

[webView loadRequest:request];
```

b、通过 document.cookie 设置 Cookie 解决后续页面(同域)Ajax、iframe 请求的 Cookie 问题；

注意：document.cookie()无法跨域设置 cookie
```objc
WKUserContentController* userContentController = [WKUserContentController new];

WKUserScript * cookieScript = [[WKUserScript alloc] initWithSource: @"document.cookie = 'skey=skeyValue';" injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];

[userContentController addUserScript:cookieScript];
```

这种方案无法解决302请求的 Cookie 问题，比如，第一个请求是 www.a.com，我们通过在 request header 里带上 Cookie 解决该请求的 Cookie 问题，接着页面302跳转到 www.b.com，这个时候 www.b.com 这个请求就可能因为没有携带 cookie 而无法访问。当然，由于每一次页面跳转前都会调用回调函数：
```objc
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void(^)(WKNavigationActionPolicy))decisionHandler;
```

可以在该回调函数里拦截302请求，copy request，在 request header 中带上 cookie 并重新 loadRequest。不过这种方法依然解决不了页面 iframe 跨域请求的 Cookie 问题，毕竟-[WKWebView loadRequest:]只适合加载 mainFrame 请求。

### 3、WKWebView NSURLProtocol问题

WKWebView 在独立于 app 进程之外的进程中执行网络请求，请求数据不经过主进程，因此，在 WKWebView 上直接使用 NSURLProtocol 无法拦截请求。苹果开源的 webKit2 源码暴露了 [私有API](https://www.jianshu.com/p/bb20ff351fa2) ：

```objc
+ [WKBrowsingContextController registerSchemeForCustomProtocol:]
```

通过注册 http(s) scheme 后 WKWebView 将可以使用 NSURLProtocol 拦截 http(s) 请求：

```objc
Class cls = NSClassFromString(@"WKBrowsingContextController”);

SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");

if ([(id)cls respondsToSelector:sel]) {

	// 注册http(s) scheme, 把 http和https请求交给 NSURLProtocol处理
	
	[(id)cls performSelector:sel withObject:@"http"];
	
	[(id)cls performSelector:sel withObject:@"https"];

}
```

**但是这种方案目前存在两个严重缺陷：**

a、post 请求 body 数据被清空

由于 WKWebView 在独立进程里执行网络请求。一旦注册 http(s) scheme 后，网络请求将从 Network Process 发送到 App Process，这样 NSURLProtocol 才能拦截网络请求。在 webkit2 的设计里使用 MessageQueue 进行进程之间的通信，Network Process 会将请求 encode 成一个 Message,然后通过 IPC 发送给 App Process。出于性能的原因，encode 的时候 HTTPBody 和 HTTPBodyStream 这两个字段被丢弃掉了

参考苹果源码：

[https://github.com/WebKit/webkit/blob/fe39539b83d28751e86077b173abd5b7872ce3f9/Source/WebKit2/Shared/mac/WebCoreArgumentCodersMac.mm#L61-L88](https://www.jianshu.com/p/bb20ff351fa2) 

及bug report:

[https://bugs.webkit.org/show_bug.cgi?id=138169](https://www.jianshu.com/p/bb20ff351fa2) 

因此， **如果通过 registerSchemeForCustomProtocol 注册了 http(s) scheme, 那么由 WKWebView 发起的所有 http(s)请求都会通过 IPC 传给主进程 NSURLProtocol 处理，导致 post 请求 body 被清空** ；

b、对ATS支持不足

测试发现一旦打开ATS开关： **Allow Arbitrary Loads 选项设置为NO** ，同时通过 registerSchemeForCustomProtocol 注册了 http(s) scheme，WKWebView 发起的所有 http 网络请求将被阻塞（即便将 **Allow Arbitrary Loads in Web Content 选项设置为YES** ）；

WKWebView 可以注册 customScheme, 比如 dynamic://, 因此希望使用离线功能又不使用 post 方式的请求可以通过 customScheme 发起请求，比如 dynamic://www.dynamicalbumlocalimage.com/，然后在 app 进程 NSURLProtocol 拦截这个请求并加载离线数据。不足：使用 post 方式的请求该方案依然不适用，同时需要 H5 侧修改请求 scheme 以及 CSP 规则；

### 4、WKWebView loadRequest 问题

在 WKWebView 上通过 loadRequest 发起的 post 请求 body 数据会丢失：

//同样是由于进程间通信性能问题，HTTPBody字段被丢弃

```objc
[request setHTTPMethod:@"POST"];
[request setHTTPBody:[@"bodyData"dataUsingEncoding:NSUTF8StringEncoding]];
[wkwebview loadRequest: request];
```

workaround:

假如想通过-[WKWebView loadRequest:]加载 post 请求 request1:**[http://h5.qzone.qq.com/mqzone/index](https://www.jianshu.com/p/bb20ff351fa2)**,可以通过以下步骤实现：

替换请求 scheme，生成新的 post 请求 request2:**post://h5.qzone.qq.com/mqzone/index**, 同时将 request1 的 body 字段复制到 request2 的 header 中（WebKit 不会丢弃 header 字段）;

通过-[WKWebView loadRequest:]加载新的 post 请求 request2;

通过 +[WKBrowsingContextController registerSchemeForCustomProtocol:]注册 scheme:**post://**;

注册 NSURLProtocol 拦截请求 **post://h5.qzone.qq.com/mqzone/index**,替换请求 scheme, 生成新的请求 request3:**[http://h5.qzone.qq.com/mqzone/index](https://www.jianshu.com/p/bb20ff351fa2)** ，将 request2 header的body 字段复制到 request3 的 body 中，并使用 NSURLConnection 加载 request3，最后通过 NSURLProtocolClient 将加载结果返回 WKWebView;

### 5、WKWebView 页面样式问题

在 WKWebView 适配过程中，我们发现部分H5页面 **元素位置向下偏移** 或 **被拉伸变形** ，追踪后发现主要是H5页面高度值异常导致：

**a**. 空间H5页面有透明导航、透明导航下拉刷新、全屏等需求，因此之前 webView 整个是从（0, 0）开始布局，通过调整webView.scrollView.contentInset来适配特殊导航栏需求。而在 WKWebView 上对 contentInset 的调整会反馈到webView.scrollView.contentSize.height的变化上，比如设置webView.scrollView.contentInset.top = a，那么contentSize.height的值会增加a,导致H5页面长度增加，页面元素位置向下偏移；

解决方案是： **调整WKWebView布局方式，避免调整webView.scrollView.contentInset** 。实际上，即便在 UIWebView 上也不建议直接调整webView.scrollView.contentInset的值，这确实会带来一些奇怪的问题。如果某些特殊情况下非得调整 contentInset 不可的话，可以通过下面方式让H5页面恢复正常显示：
```objc

/**设置contentInset值后通过调整webView.frame让页面恢复正常显示

*参考：http://km.oa.com/articles/show/277372

*/

webView.scrollView.contentInset = UIEdgeInsetsMake(a,0,0,0); 
webView.frame = CGRectMake(webView.frame.origin.x, webView.frame.origin.y, webView.frame.size.width, webView.frame.size.height - a);

```

**b**. 在接入 now 直播的时候，我们发现在 iOS 9 上 WKWebView 会出现页面被拉伸变形的情况，最后发现是window.innerHeight值不准确导致（在WKWebView上返回了一个非常大的值），而H5同学通过获取window.innerHeight来设置页面高度，导致页面整体被拉伸。通过查阅相关资料发现，这个bug只在 iOS 9 的几个系统版本上出现，苹果后来fix了这个bug。我们最后的解决方案是： ***延迟调用window.innerHeight***

```objc
setTimeout(function(){height = window.innerHeight},0);

or

Use shrink-to-fit meta-tag
```

### 6、WKWebView 截屏问题

空间玩吧H5小游戏有截屏分享的功能，WKWebView 下通过 -[CALayer renderInContext:]实现截屏的方式失效，需要通过以下方式实现截屏功能：

```objc
@implementationUIView (ImageSnapshot) 
- (UIImage*)imageSnapshot {     
	UIGraphicsBeginImageContextWithOptions(self.bounds.size,YES,self.contentScaleFactor);     
	[self drawViewHierarchyInRect:self.bounds afterScreenUpdates:YES];     
	UIImage* newImage = UIGraphicsGetImageFromCurrentImageContext();     
	UIGraphicsEndImageContext();returnnewImage; 
}
@end
```

然而这种方式依然解决不了 webGL 页面的截屏问题，笔者已经翻遍苹果文档，研究过 webKit2 源码里的 [截屏私有API](https://www.jianshu.com/p/bb20ff351fa2) ，依然没有找到合适的解决方案，同时发现 Safari 以及 Chrome 这两个全量切换到 WKWebView 的浏览器也存在同样的问题： **对webGL 页面的截屏结果不是空白就是纯黑图片** 。无奈之下，我们只能约定一个JS接口，让游戏开发商实现该接口，具体是通过canvas getImageData()方法取得图片数据后返回 base64 格式的数据，客户端在需要截图的时候，调用这个JS接口获取 base64 String 并转换成 UIImage。

### 7、WKWebView crash问题

WKWebView 放量后，外网新增了一些 crash, 其中一类 crash 的主要堆栈如下：

```js
...28UIKit0x0000000190513360UIApplicationMain +208

29Qzone0x0000000101380570main (main.m:181)30libdyld.dylib0x00000001895205b8_dyld_process_info_notify_release +36

Completion handler passed to -[QZWebController webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:] was not called
```

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

如果 WKWebView 退出的时候，JS刚好执行了window.alert(), alert 框可能弹不出来，completionHandler 最后没有被执行，导致 crash；另一种情况是在 WKWebView 一打开，JS就执行window.alert()，这个时候由于 WKWebView 所在的 UIViewController 出现（push或present）的动画尚未结束，alert 框可能弹不出来，completionHandler 最后没有被执行，导致 crash。我们最终的实现大致是这样的：

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

另一个 crash 发生在 WKWebView 退出前调用：

```objc
-[WKWebView evaluateJavaScript: completionHandler:]
```

执行JS代码的情况下。WKWebView 退出并被释放后导致completionHandler变成野指针，而此时 javaScript Core 还在执行JS代码，待 javaScript Core 执行完毕后会调用completionHandler()，导致 crash。这个 crash 只发生在 iOS 8 系统上，参考Apple Open Source，在iOS9及以后系统苹果已经修复了这个bug，主要是对completionHandler block做了copy（refer:[https://trac.webkit.org/changeset/179160](https://www.jianshu.com/p/bb20ff351fa2) ）；对于iOS 8系统，可以通过在 completionHandler 里 retain WKWebView 防止 completionHandler 被过早释放。我们最后用 methodSwizzle hook 了这个系统方法：

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

### 8、其它问题

8.1、视频自动播放

WKWebView 需要通过WKWebViewConfiguration.mediaPlaybackRequiresUserAction设置是否允许自动播放，但一定要在 WKWebView 初始化之前设置，在 WKWebView 初始化之后设置无效。

8.2、goBack API问题

WKWebView 上调用 -[WKWebView goBack], 回退到上一个页面后不会触发window.onload()函数、不会执行JS。

8.3、页面滚动速率

WKWebView 需要通过scrollView delegate调整滚动速率：

```objc
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView {
	scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
}
```

9、结语

本文总结了在 WKWebView 上踩过的一些坑。虽然 WKWebView 坑比较多，但是相对 UIWebView 在内存消耗、稳定性方面还是有很大的优势。尽管苹果对 WKWebView 的开发进度过于缓慢，但相信 WKWebView 才是未来。

[WKWebView 那些坑(转自 腾讯Bugly) - 简书](https://www.jianshu.com/p/bb20ff351fa2)