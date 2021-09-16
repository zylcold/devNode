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