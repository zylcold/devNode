#webkit/issus #workaround 

# 表现
在 WKWebView 适配过程中，我们发现部分H5页面 **元素位置向下偏移** 或 **被拉伸变形** 

# 原因
主要是H5页面高度值异常导致：
**a**. 空间H5页面有透明导航、透明导航下拉刷新、全屏等需求，因此之前 webView 整个是从（0, 0）开始布局，通过调整webView.scrollView.contentInset来适配特殊导航栏需求。而在 WKWebView 上对 contentInset 的调整会反馈到webView.scrollView.contentSize.height的变化上，比如设置webView.scrollView.contentInset.top = a，那么contentSize.height的值会增加a,导致H5页面长度增加，页面元素位置向下偏移；

**b**. 在接入 now 直播的时候，我们发现在 iOS 9 上 WKWebView 会出现页面被拉伸变形的情况，最后发现是window.innerHeight值不准确导致（在WKWebView上返回了一个非常大的值），而H5同学通过获取window.innerHeight来设置页面高度，导致页面整体被拉伸。通过查阅相关资料发现，这个bug只在 iOS 9 的几个系统版本上出现，苹果后来fix了这个bug。

# 解决方案

A:  调整WKWebView布局方式，避免调整webView.scrollView.contentInset 。实际上，即便在 UIWebView 上也不建议直接调整webView.scrollView.contentInset的值，这确实会带来一些奇怪的问题。如果某些特殊情况下非得调整 contentInset 不可的话，可以通过下面方式让H5页面恢复正常显示：
```objc

/**设置contentInset值后通过调整webView.frame让页面恢复正常显示

*参考：http://km.oa.com/articles/show/277372

*/

webView.scrollView.contentInset = UIEdgeInsetsMake(a,0,0,0); 
webView.frame = CGRectMake(webView.frame.origin.x, webView.frame.origin.y, webView.frame.size.width, webView.frame.size.height - a);

```

B: 延迟调用window.innerHeight

```objc
setTimeout(function(){height = window.innerHeight},0);

//or

Use shrink-to-fit meta-tag
```