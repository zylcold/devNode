#webkit/issus 

# 表现
WKWebView 上调用 `-[WKWebView goBack]`, 回退到上一个页面后不会触发`window.onload()`函数、不会执行JS。