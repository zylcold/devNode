# 简介
WKWebView 是苹果在 WWDC 2014 上推出的新一代 webView 组件，用以替代 UIKit 中笨重难用、内存泄漏的 UIWebView。

# 与UIWebview的对比
![[WKWebView和UIWebView内核引擎的区别#优点]]
![[WKWebView和UIWebView内核引擎的区别#问题]]
详细[[WKWebView和UIWebView内核引擎的区别]]

# 拦截
![[WKWebView 请求拦截探索与实践#方案调研]]
详细[[WKWebView 请求拦截探索与实践]]

# 与JS的交互
- [[执行网页的JS代码(Native->JS)]]
- [[网页注入JS代码(Native->JS)]]
- [[使用MessageHandler交互(JS->Native)]]
- [[通过cookies向网页传值(Native->JS)]]
- [[JS同步调用原生代码(JS->Native)]]