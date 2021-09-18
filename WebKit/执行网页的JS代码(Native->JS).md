* 直接执行JS代码
执行JS方法，使用 `evaluateJavaScript` 方法即可，一行代码搞定:

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

```swift
//swift代码
let js ="window.toggleCatalog();" 
webView?.evaluateJavaScript(js, completionHandler: nil)
```

一般使用以上方法,是在网页加载完成( `didFinishNavigationv` )的时候进行操作.

注意：[[WKWebView evaluateJavaScript引起的crash]]


# 传值
## 通过JS方法传值
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

## 通过window变量传值
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