#webkit/issus  #workaround 
# 表现
视频无法自动播放

# 解决方案
通过WKWebViewConfiguration.mediaPlaybackRequiresUserAction设置是否允许自动播放，但一定要**在 WKWebView 初始化之前设置**，在 WKWebView 初始化之后设置无效。
