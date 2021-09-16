# 表现
WKWebView 存储时机有延迟，导致当前页面无法实时拿到Cookie
Cookie 问题是目前 WKWebView 的一大短板

# 原因
业界普遍认为 WKWebView 拥有自己的私有存储，不会将 Cookie 存入到标准的 Cookie 容器 **NSHTTPCookieStorage** 中。

实践发现 WKWebView 实例其实也会将 Cookie 存储于 NSHTTPCookieStorage 中，但存储时机有延迟，在iOS 8上，当页面跳转的时候，当前页面的 Cookie 会写入 NSHTTPCookieStorage 中，而在 iOS 10 上，JS 执行 document.cookie 或服务器 set-cookie 注入的 Cookie 会很快同步到 NSHTTPCookieStorage 中，FireFox 工程师曾建议通过 reset WKProcessPool 来触发 Cookie 同步到 NSHTTPCookieStorage 中，实践发现不起作用，并可能会引发当前页面 session cookie 丢失等问题。

**WKWebView Cookie 问题在于 WKWebView 发起的请求不会自动带上存储于 NSHTTPCookieStorage 容器中的 Cookie** 。

比如，NSHTTPCookieStorage 中存储了一个 Cookie:
```js
name=Nicholas;value=test;domain=y.qq.com;expires=Sat,02May201923:38:25GMT；
```

通过 UIWebView 发起请求 [http://y.qq.com，](https://www.jianshu.com/p/bb20ff351fa2) 则请求头会自动带上 cookie: Nicholas=test；

而通过 WKWebView发起请求 [http://y.qq.com，](https://www.jianshu.com/p/bb20ff351fa2) 请求头不会自动带上 cookie: Nicholas=test。

2.2、WKProcessPool

苹果开发者文档对 WKProcessPool 的定义是： [A WKProcessPool object represents a pool of Web Content process](https://www.jianshu.com/p/bb20ff351fa2). 通过让所有 WKWebView 共享同一个 WKProcessPool 实例，可以 **实现多个 WKWebView 之间共享 Cookie（session Cookie and persistent Cookie）数据** 。不过 WKWebView WKProcessPool 实例在 app 杀进程重启后会被重置，导致 WKProcessPool 中的 Cookie、session Cookie 数据丢失，目前也无法实现 WKProcessPool 实例本地化保存。

# 解决方案

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