#webview 

[XueYongWei](https://www.jianshu.com/u/0c5efe86aa33)

#### 基本应用

首先使用WKWebView.你需要导入WebKit。关于WKWebView其他基础使用不在本篇研究范围。
博主不才，本文根据实际做项目过程中所运用到的进行总结。有不足及错误的地方，还请在评论里指教。本文中所列代码及论述的方法，都在实际项目中应用过的，此项目包含完整的浏览器功能，运用到的模块有：网页进入看图模式、长按链接选择新窗口打开、网页夜间模式、115离线下载、网页原生互相跳转、本公司网页免登录、经验号网页免登录点赞收藏、获取经验网页上文章作者用户信息等。

User-agent:[为什么所有浏览器的User-agent总是有Mozilla？](https://www.jianshu.com/p/4cfca5ae9c0f)

#### 目录索引

`本文内容稍杂，由于简书不支持锚点，可以通过复制目录摘要进行搜索，找到自己想要看的内容。`
本篇讨论WKWebview的综合使用，包括以下几个部分：

* JS交互与数据传递。

	* iOS端执行JS

		* iOS端执行JS代码字符串
		* iOS端执行网页JS方法

	* JS执行iOS端方法

		* 通过iOS原生API执行
		* 跨平台通用方法执行

	* iOS向网页注入JS
	* 网页向iOS传值
	* iOS向网页传值

		* 通过cookies传值
		* 通过JS方法传值
		* 通过window变量传值

* cookies的注入与清除。

	* cookies的注入

		* 注入cookies
		* 主动使用cookies

	* cookies的清除

		* 清除全部cookies
		* 清除某域名下的cookies
		* iOS9之前版本清除cookies

#### ======== WKWebview ========

#### 一. JS交互与数据传递

我们使用WKWebview进行更复杂的用法时，难免遇到需要和JS进行交互的情况，方法的相互调用，数据的传递等。

###### 1. iOS端执行JS

* 直接执行JS代码
执行JS方法，使用 `evaluateJavaScript` 方法即可，一行代码搞定:

```objc
//OC代码
[webView evaluateJavaScript:jsStr completionHandler:^(id _Nullable response, NSError * _Nullable error) {
}];
```

```swift
//swift代码
webview.evaluateJavaScript(jsStr) { (response, error) in

											 
}
```

比如执行JS代码获取网页的标题:

```javascript
webView.evaluateJavaScript("document.title") { (jsstr, error) in
	if let t = jsstr as? String{
	  self.title = t
	}
}
```

* 执行网页的JS方法
* 当然，你还有一个使用场景，想要原生某些控件去调用网页里已有的某个js方法。比如：某个网页里有点击到达顶部，或者显示网页章节目录等js方法，我们针对这个网页在原生空间做特殊处理：点击原生toolBar上的目录按钮，让网页显示出章节目录。我们只需要直接执行这个js方法即可。
* 已知js的显示章节目录的js方法名为 `toggleCatalog`,代码如下：

```objc
//OC代码
NSString *jsStr  = @"window.toggleCatalog();";
[self.webView evaluateJavaScript:jsStr completionHandler:^(id _Nullable, NSError * _Nullable error) {
        
}]
```

```swift
//swift代码
let js ="window.toggleCatalog();" 
webView?.evaluateJavaScript(js, completionHandler: nil)
```

一般使用以上方法,是在网页加载完成( `didFinishNavigationv` )的时候进行操作.

###### 2. JS执行iOS端方法

* 通过iOS原生API执行
* JS执行iOS端方法，调用
* `window.webkit.messageHandlers.<对象名>.postMessage(<数据>)`
* 方法，上方代码 `在JS端写会报错,导致页面后面业务不执行。可使用try-catch执行。`
* 当然，JS调用的方法，必须是和客户端约定好的，在iOS中的处理方法是：

```objc
- (void)addScriptMessageHandler:(id <WKScriptMessageHandler>)scriptMessageHandler name:(NSString *)name;
```

它是WKScriptMessageHandler的代理方法.name和上方JS中的对象名相对应.

* 跨平台通用方法执行
* 在实际应用中，因为h5的跨平台因素，为了和其他平台js方法使用保持一致，我们最好自己写个纯粹的js方法，在方法里调用iOS下的 `window.webkit.messageHandlers.<对象名>.postMessage(<数据>)` 。
比如JS需要APP打开用户登录，我们写一个JS方法：

```javascript
//我们往网页里注入的js代码
function callLogin() {
	// APPJS是我们所注入的对象
	window.webkit.messageHandlers.APPJS.postMessage("shouldLogin");
}
```

这样注入网页后，网页JS只需调用 `callLogin()` 即可唤起原生的方法。
原生代码里处理方式如下：

```objc
//OC代码
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
    
	if ([message.name isEqualToString:@"APPJS"]) {
        NSLog(@"%@", message.body);
   }
}
```

一般的，我们可以将需要调用的方法写入一个js文件，然后注入到网页，网页就可以直接调用这个js方法。

###### 2. iOS向网页注入JS

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

###### 3. 网页向iOS传值

刚才我们已经试过了最简单的放法，但是我们可能需要向原生传值，比如网页加载一片文章完毕，需要告诉原生app，我网页上的一些用户信息。
我们约定了一个方法叫 `setPageInfo(info)`,info是json类型。

```javascript
//复杂点的js方法,参数中约定好格式。
//比如：fun代表方法名，arg代表参数
function setPageInfo(info) {
	// APPJS是我们所注入的对象
	window.webkit.messageHandlers.APPJS.postMessage({fun: 'setPageInfo',
       arg: {
         pageInfo: {
           userID:10086,
           userName:'中国联通',
           isFav:false,
         }
       }
   });
}
```

网页只需要调用此方法，我们原生就能得到网页给我们的数据，iOS端处理如下。

```objc
//OC代码
- (void)userContentController:(WKUserContentController *)userContentController
      didReceiveScriptMessage:(WKScriptMessage *)message {
    if ([message.name isEqualToString:@"APPJS"]) {
        // 打印所传过来的参数，只支持NSNumber, NSString, NSDate, NSArray,
        // NSDictionary, and NSNull类型
        //这里message.body就是JS传过来的info，按字典取值即可。判断fun和args以特殊处理。
        NSLog(@"%@", message.body);
    }else if ([message.name isEqualToString:@"AppModel"]){
        NSLog(@"%@", message.body);
    }
}
```

在swift下我们接收、处理数据。

```swift
//swift代码,这个对应上面比较复杂js的处理，按约定格式
func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage){
     if message.name == "APPJS" {
		  if let dic = message.body as? NSDictionary, dic["fun"] != nil,
				let fun = (dic["fun"] as AnyObject).description{
                if let arg = dic["arg"] as? NSArray {
                    if fun == "setPageInfo" {
                        //去取数据即可。
                  	}
              }
     }
}
```

###### 3. iOS向网页传值

* 通过cookies传值
* 查看第二部分： `二. cookies的注入与清除。`
* 我们通过设置cookies，然后与网页开发者约定，让网页从cookies里取值即可。（比如用户的authKey）
* 通过JS方法传值
* 实际上，在我们使用JS调用原生方法的时候，应该就会有这个疑问：如果获得返回值？是的，暂时没办法获得返回值。
* 不过我们可以曲线救国。
* 比如我们需要获得app的用户信息：

```javascript
getUserInfoString: function () {
  window.webkit.messageHandlers.OOFJS.postMessage({
    fun: 'getUserInfoString',
    arg: {
    callback: 'setJSUserInfo'
    }
  })
},
```

注意到我们的arg参数里，多了个参数叫 `callback: 'setJSUserInfo'` ，那么我们就知道我们收集完用户信息，怎么告诉JS用户信息是什么。
我们只需要通过第一步的方法，执行这个JS方法，并传入用户信息即可。

```swift
let infoStr = getUserInfoJsonStr()
let jsStr =  "setJSUserInfo(infoStr)"
webView?.evaluateJavaScript(, completionHandler: nil)
```

然后JS的这个方法就被回调了，JS可以在得到用户信息后，做进一步的数据更新。

* 通过window变量传值
* 通过第二步我们已经能正确的让JS拿到我们iOS里面的信息，但是我们发现不能及时地拿到返回值会有很大的限制，比如有这种情况，JS需要直接拿到返回值进行其他操作：

```javascript
//JS代码
var info = getUserInfoString()
if info....
```

这就GG了，因为JS调用这个方法的时候，我们没办法直接给它返回。
我们再次曲线救国。
我们可以在调用这个方法之前，比如网页加载完毕，网页收到相应等情况下，提前准备好数据，然后写入到window里！把infoString写入window的方法：

```swift
let infoStr = getUserInfoJsonStr()
let jsStr =  "window.GLOBAL_USERINFO=\(infoStr)"
webView.evaluateJavaScript(jsStr, completionHandler: nil)
```

执行完毕后，window就有了一个GLOBAL_USERINFO变量，随时可取。不过网页开发者不必知道这个东西，无需为了iOS而特殊处理，仍让他调用 `getUserInfoString` 方法即可。
然后我们改写这个方法，让它能直接返回infoStr：

```javascript
getUserInfoString: function () {
  window.webkit.messageHandlers.OOFJS.postMessage({
    fun: 'getUserInfoString',
    arg: {
    }
  })
  return window.GLOBAL_USERINFO;
},
```

这个方法是直接返回了我们提前写入的变量window.GLOBAL_USERINFO。

#### 二. cookies的注入与清除。

如果你的App是从iOS11开始，那么仅仅需要这么做：

1. 获取cookie

```objc
WKHTTPCookieStore *cookieStore = self.webView.configuration.websiteDataStore.httpCookieStore;
[cookieStore getAllCookies:^(NSArray<NSHTTPCookie *> * _Nonnull cookies) {
      NSLog(@"All cookies %@",cookies);
}];
```

1. 注入cookie

```objc
WKHTTPCookieStore *cookieStore = self.webiew.configuration.websiteDataStore.httpCookieStore;
[cookieStore setCookie:cookie completionHandler:nil];
```

注入的cookie也是长期的。
如果你的App是兼容更早版本的，那么就麻烦点了。

* cookie注入
* cookie相关，特别是需要兼容iOS9之前，会有较多的坑。 [推荐一篇比较好的博客](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.skyfox.org%2Fios-wkwebview-cookie-opration.html)
> WKWebView会忽视默认的网络存储， NSURLCache, NSHTTPCookieStorage, NSCredentialStorage。 目前是这样的，WKWebView有自己的进程，同样也有自己的存储空间用来存储cookie和cache， 其他的网络类如NSURLConnection是无法访问到的。 同时WKWebView发起的资源请求也是不经过NSURLProtocol的，导致无法自定义请求。

WKWebView与UIWebview的一个区别，就是WKWebView实例将会忽略任何的默认网络存储器(NSURLCache, NSHTTPCookieStorage, NSCredentialStorage) 和一些标准的自定义网络请求类(NSURLProtocol,等等.).
WKWebView实例不会把Cookie存入到App标准的的Cookie容器(NSHTTPCookieStorage)中,因为 NSURLSession/NSURLConnection等网络请求使用NSHTTPCookieStorage进行访问Cookie,所以不能访问WKWebView的Cookie,现象就是WKWebView存了Cookie,其他的网络类如NSURLSession/NSURLConnection却看不到。
与Cookie相同的情况就是WKWebView的缓存,凭据等。WKWebView都拥有自己的私有存储,因此和标准cocoa网络类兼容的不是那么好

> NSHTTPCookieStorage 实现管理cookie的单利，每个cookie都是NSHTTPCookie类的实例，做为一个规则，cookie在所有应用 之间共享并在不同进程之间保持同步。
> 上面引入了网页需要用户登陆，然后让app跳转登陆界面进行登陆，app登陆之后自然要向网页注入cookie，来让网页继续剩下的功能。

###### 1. 在webview发起请求的时候附带cookie。

这个适用首次发起网页请求，同样适用点击，在webview代理方法里，判断是否需要注入cookie的域名，如果是，截断请求，重新发起注入了cookie的请求。

```objc
//oc代码
NSMutableDictionary *cookieDic = [NSMutableDictionary dictionary];
NSMutableString *cookieValue = [NSMutableString stringWithFormat:@""];
NSHTTPCookieStorage *cookieJar = [NSHTTPCookieStorage sharedHTTPCookieStorage];
for (NSHTTPCookie *cookie in [cookieJar cookies]) {
     [cookieDic setObject:cookie.value forKey:cookie.name];
}
    
 // cookie重复，先放到字典进行去重，再进行拼接
for (NSString *key in cookieDic) {
      NSString *appendString = [NSString stringWithFormat:@"%@=%@;", key, [cookieDic valueForKey:key]];
      [cookieValue appendString:appendString];
}
NSMutableURLRequest * request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:self.url]];
[request addValue:cookieValue forHTTPHeaderField:@"Cookie"];
NSLog(@"添加cookie");
[self.webView loadRequest:request];
```

```swift
//swift代码
guard let cookies = HTTPCookieStorage.shared.cookies else {
            return
 }
var cookieDic = Dictionary<String, Any>()
var cookieValue = ""
for cookie in  cookies{
   cookieDic[cookie.name] = cookie.value
}
for (key,value) in cookieDic {
   let appendString = "\(key)=\(value)"
   cookieValue.append(appendString)
}
 let request = URLRequest.init(url: URL.init(string: "url")!)
 request.addValue(cookieValue, forHTTPHeaderField: "Cookie")
```

###### 2. 在webview创建的时候js注入cookie。

其中js的写法问题，有可能有多个写法是cookie之间用 `；` 隔开。

```objc
//OC代码
WKUserContentController* userContentController = WKUserContentController.new;

WKUserScript * cookieScript = [[WKUserScript alloc] initWithSource: @"document.cookie ='TeskCookieKey1=TeskCookieValue1';document.cookie = 'TeskCookieKey2=TeskCookieValue2';"injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];

[userContentController addUserScript:cookieScript];

WKWebViewConfiguration* webViewConfig = WKWebViewConfiguration.new;

webViewConfig.userContentController = userContentController;

WKWebView * webView = [[WKWebView alloc] initWithFrame:CGRectMake(/*set your values*/) configuration:webViewConfig];
```

```swift
//swift代码
let userContent = WKUserContentController()
let jsStr = "document.cookie ='TeskCookieKey1=TeskCookieValue1';document.cookie = 'TeskCookieKey2=TeskCookieValue2';"
        
let cookieScript = WKUserScript.init(source: jsStr, injectionTime: WKUserScriptInjectionTime.atDocumentStart, forMainFrameOnly: false)
userContent.addUserScript(cookieScript)
let webViewConfig = WKWebViewConfiguration()
webViewConfig.userContentController = userContent
        
let webview = WKWebView.init(frame: CGRect(x: 0, y: 0, width: 300, height: 300), configuration: webViewConfig)
```

###### 3. 在webview加载内容时js注入cookie。

```swift
//swift代码
func webView(_ webView: WKWebView, didCommit navigation: WKNavigation!) {
if let laToken = UserCenter.shared().user?.laToken {
                let cookie = "115token=\(oofToken)"
                webView?.evaluateJavaScript("function setCookie(e,o){document.cookie=e+\"=\"+escape(o)+\";path=/;domain=.115.com\"}for(var cookieTem= \"\(cookie)\",cookieArr=cookieTem.split(\";\"),i=0;i<cookieArr.length;i++){var temArr=cookieArr[i].split(\"=\");setCookie(temArr[0],temArr[1])}", completionHandler: {
                    (object, error) -> Void in
                })
            }
}
```

经过验证，最好是第一点和第三点同时使用，第二点每次截断请求总觉得浪费资源 -。-

* 主动使用cookies
* 通过写入或修改cookies，可以达到与网页开发者商定的数据交流。
###### 2. cookie清除

某些情况，需要清理已经注入的cookie，比如浏览器的清理缓存，或者用户退出登录等。需要注意的是WKWebview的清理cookieAPI，是iOS9之后才有的。

* 清除全部cookies
* WKWebsiteDataStore可根据需要自行选择，具体参数参阅文档与注释。

```swift
let websiteDataTypes = WKWebsiteDataStore.allWebsiteDataTypes()
            
let dateFrom = Date.init(timeIntervalSince1970: 0)
            
WKWebsiteDataStore.default().removeData(ofTypes: websiteDataTypes, modifiedSince: dateFrom, completionHandler: {
})
```

注意：此操作将会清空cookies，类似浏览器里“清除记录”的功能。

* 清除某域名下的cookies
* 如果是app的用户退出登录，需要清理的仅仅是自己家cookie(即某域名下)，则可以这样:
* 比如我们现在app里退出登录，需要清理自己用户的的数据，而不是全部cookies，代码如下：
* （displayName是域名，我们这里是只要包含115就清理。）

```swift
WKWebsiteDataStore.default().fetchDataRecords(ofTypes:websiteDataTypes, completionHandler: { (records) in
  for record in records {
    debugPrint("fetch cookies -> \(record)")
    if record.displayName.contains("115") {
      WKWebsiteDataStore.default().removeData(ofTypes: WKWebsiteDataStore.allWebsiteDataTypes(), for: [record], completionHandler: {
      debugPrint("清理了cookies -> \(record)")
      })
    }
   }
})
```

* iOS9之前版本清除cookies
* 在iOS9之前，wkwebview是没有清理cookie的方法的，所以需要对不同的版本进行不同的操作。
* 那iOS9之前的如何操作？可以预见，既然是缓存，肯定是放在沙盒里的。找到沙盒的目录，删除文件即可。

```swift
/// 清理cookie缓存数据
func ClearCache() {
  if #available(iOS 9.0, *) {
    // 根据需求，通过API清理cookie
  } else {//否则清理文件夹
    let libraryPath = NSSearchPathForDirectoriesInDomains(.libraryDirectory, .userDomainMask, true)[0]
    let cookiesFolderPath = libraryPath+"/Cookies"
    try? FileManager.default.removeItem(atPath: cookiesFolderPath)
  }
}
```


https://www.jianshu.com/p/c2a09a057306