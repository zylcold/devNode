#webkit/issus #workaround

# 表现
当内存占用太大的时候，在 UIWebView 上，App Process 会 crash；而在 WKWebView，WebContent Process 会 crash，从而出现白屏现象。  

在 WKWebView 中加载下面的测试链接可以稳定重现白屏现象:  

[http://people.mozilla.org/~rnewman/fennec/mem.html](https://www.jianshu.com/p/bb20ff351fa2)

这个时候**WKWebView.URL会变为nil**, 简单的 reload 刷新操作已经失效，对于一些长驻的H5页面影响比较大。  

# 原因
WKWebView 自诩拥有更快的加载速度，更低的内存占用，但实际上 WKWebView 是一个多进程组件，Network Loading 以及 UI Rendering 在其它进程中执行。详细[[WKWebView和UIWebView内核引擎的区别#多进程]] 		

初次适配 WKWebView 的时候，我们也惊讶于打开 WKWebView 后，App 进程内存消耗反而大幅下降，但是仔细观察会发现，Other Process 的内存占用会增加。在一些用 webGL 渲染的复杂页面，使用 WKWebView 总体的内存占用（App Process Memory + Other Process Memory）不见得比 UIWebView 少很多。

# 解决方案

A、借助 **WKNavigtionDelegate**

iOS 9以后 WKNavigtionDelegate 新增了一个回调函数：
```objc
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView API_AVAILABLE(macosx(10.11),ios(9.0));
```

当 WKWebView 总体内存占用过大，**页面即将白屏**的时候，系统会调用上面的回调函数，我们在该函数里执行`[webView reload]`(这个时候 webView.URL 取值尚不为 nil）解决白屏问题。在一些高内存消耗的页面可能会频繁刷新当前页面，H5侧也要做相应的适配操作。

B、检测 webView.title 是否为空

并不是所有H5页面白屏的时候都会调用上面的回调函数，比如，最近遇到在一个高内存消耗的H5页面上 present 系统相机，拍照完毕后返回原来页面的时候出现白屏现象（拍照过程消耗了大量内存，导致内存紧张，WebContent Process 被系统挂起），但上面的回调函数并没有被调用。在WKWebView白屏的时候，另一种现象是 webView.titile 会被置空, 因此，可以在 viewWillAppear 的时候检测 webView.title 是否为空来 reload 页面。

综合以上两种方法可以解决绝大多数的白屏问题。
