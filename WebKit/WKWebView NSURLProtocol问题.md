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