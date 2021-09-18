* 我们通过设置cookies，然后与网页开发者约定，让网页从cookies里取值即可。（比如用户的authKey）
### 二. cookies的注入与清除。

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

### 1. 在webview发起请求的时候附带cookie。

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

### 2. 在webview创建的时候js注入cookie。

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

### 3. 在webview加载内容时js注入cookie。

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

## 2. cookie清除

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
