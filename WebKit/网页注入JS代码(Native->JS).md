通过使用userContentController向网页注入JS，注入的JS可以取名字，将会在 `WKScriptMessageHandler` 的代理方法 `didReceiveScriptMessage` 中被回掉。
注入的String的JS代码，简单的直接写，复杂的可写到一个JS文件里，然后读取文本，创建WKUserScript。

```objc
//OC代码
NSString *js = @"I am JS Code";
//初始化WKUserScript对象
//WKUserScriptInjectionTimeAtDocumentEnd为网页加载完成时注入

WKUserScript *script = [[WKUserScript alloc] initWithSource:js injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:YES];

//根据生成的WKUserScript对象，初始化WKWebViewConfiguration
WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
[config.userContentController addUserScript:script];

//设置ScriptMessageHandler为self
[config.userContentController addScriptMessageHandler:self name:@"APPJS"];
self.webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:config];
```

通过从js文件里读取js代码，然后注入网页。

```swift
//swift代码
//从js文件加载js代码
let path = (bundlePath) + ("/" + "Contents/Resources/ContextMenu.js")
let source = try! NSString(contentsOfFile: path, encoding: String.Encoding.utf8.rawValue) as String
            
let path2 = (bundlePath) + ("/" + "Contents/Resources/JSBridge.js")
let source2 = try! NSString(contentsOfFile: path2, encoding: String.Encoding.utf8.rawValue) as String
let js = source + source2
            
let userScript = WKUserScript(source: js, injectionTime: WKUserScriptInjectionTime.atDocumentStart, forMainFrameOnly: false)
 configuration!.userContentController.addUserScript(userScript)
  
//设置ScriptMessageHandler为self
configuration.userContentController.add(TabManager.sharedInstance, name: "APPJS")
let newWebView = WKWebView(frame: CGRect.zero, configuration: configuration)
 
self.webView = newWebView
```
