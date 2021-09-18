#webkit/issus #workaround

# 表现 
通过 loadRequest 发起的 post 请求 body 数据会丢失

```objc
[request setHTTPMethod:@"POST"];
[request setHTTPBody:[@"bodyData"dataUsingEncoding:NSUTF8StringEncoding]];
[wkwebview loadRequest: request];
```

# 原因
同样是由于进程间通信性能问题，HTTPBody字段被丢弃，[[WKWebView和UIWebView内核引擎的区别#多进程]]

# 解决方案

假如想通过`-[WKWebView loadRequest:]`加载 post 请求 request1: `http://h5.qzone.qq.com/mqzone/index`, 可以通过以下步骤实现：

1. 通过 `+[WKBrowsingContextController registerSchemeForCustomProtocol:]`注册 scheme:`post://`;
2. 替换请求 scheme，生成新的 post 请求 request2:`post://h5.qzone.qq.com/mqzone/index`, 同时将 request1 的 body 字段复制到 request2 的 header 中（WebKit 不会丢弃 header 字段）;
3. 通过`-[WKWebView loadRequest:]`加载新的 post 请求 request2; 
4. 注册 NSURLProtocol 拦截请求 `post://h5.qzone.qq.com/mqzone/index`,替换请求 scheme, 生成新的请求 request3: `http://h5.qzone.qq.com/mqzone/index` ，将 request2 header的body 字段复制到 request3 的 body 中，并使用 NSURLConnection 加载 request3
5. 最后通过 [[WKWebView 请求拦截探索与实践#WKURLSchemeHandler 拦截请求|NSURLProtocolClient]] 将加载结果返回 WKWebView