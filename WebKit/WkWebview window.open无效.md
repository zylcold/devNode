#webkit/issus   #workaround 

# 表现
JS调用 `window.open` 方法，不会起到作用。

# 原因
iOS的WkWebview对window.open方法进行了安全限制

# 解决方法

当触发window.open方法时，会触发代理WKUIDelegate中的
```objc
- (nullable WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures;
```
然后，我们就可以在这个方法中进行处理了。需要设置这个代理

```objc

webView.UIDelegate = self;


// 当调用window.open方法时，会掉用该代理方法
- (WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures{
	if (navigationAction.request.URL) {

		NSURL *url = navigationAction.request.URL;
		NSString *urlPath = url.absoluteString;
		if ([urlPath rangeOfString:@"https://"].location != NSNotFound || [urlPath rangeOfString:@"http://"].location != NSNotFound) {
			[[UIApplication sharedApplication] openURL:url];
		}
		
		
	}
	return nil;
	
	
}
```

参考： [http://stackoverflow.com/questions/30603671/open-a-wkwebview-target-blank-link-in-safa](http://stackoverflow.com/questions/30603671/open-a-wkwebview-target-blank-link-in-safa)