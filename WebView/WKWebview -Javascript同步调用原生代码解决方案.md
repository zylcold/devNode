#webview

[https://stackoverflow.com/questions/29249132/wkwebview-complex-communication-between-javascript-native-code/49474323#49474323](https://stackoverflow.com/questions/29249132/wkwebview-complex-communication-between-javascript-native-code/49474323#49474323)

This answer uses the idea from Nathan Brown's [answer](https://stackoverflow.com/a/39728672/5700434) above.
As far as I know, currently there is no way to return data back to javascript synchronous way. Hopefully apple will provide the solution in future release.
So hack is to intercept the prompt calls from js. Apple provided this functionality in order to show native popup design when js calls the alert, prompt etc. Now since prompt is the feature, where you show the data to user (we will exploit this as method param ) and the response from user to this prompt will be returned back to js (we'll exploit this as return data)
Only string can be returned. This happens in synchronous way.
We can implement the above idea as follows:
At the javascript end: call the swift method in the following way:

```javascript
function callNativeApp(){
	console.log("callNativeApp called");
	try {
		//webkit.messageHandlers.callAppMethodOne.postMessage("Hello from JavaScript");
		
		var type = "SJbridge";
		var name = "functionOne";
		var data = {name:"abc", role : "dev"}
		var payload = {type: type, functionName: name, data: data};
		
		var res = prompt(JSON.stringify (payload));
		
		//{"type":"SJbridge","functionName":"functionOne","data":{"name":"abc","role":"dev"}}
		//res is the response from swift method.
	
	} catch(err) {
		console.log('The native context does not exist yet');
	}
}
```

At the swift/xcode end do as follows:
Implement the protocol WKUIDelegate and then assign the implementation to WKWebviews uiDelegate property like this:
```objc
self.webView.uiDelegate = self
```
Now write this func webView to override (?) / intercept the request for prompt from javascript.

```javascript
func webView(_ webView: WKWebView, runJavaScriptTextInputPanelWithPrompt prompt: String, defaultText: String?, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping (String?) -> Void) {

if let dataFromString = prompt.data(using: .utf8, allowLossyConversion: false) {
	let payload = JSON(data: dataFromString)
	let type = payload["type"].string!
	
	if (type == "SJbridge") {
	
		let result = callSwiftMethod(prompt: payload)
		completionHandler(result)
	
	} else {
		AppConstants.log("jsi_", "unhandled prompt")
		completionHandler(defaultText)
	}
}else {
	AppConstants.log("jsi_", "unhandled prompt")
	completionHandler(defaultText)
}}
```

If you don't call the completionHandler() then js execution will not proceed. Now parse the json and call appropriate swift method.

```objc
func callSwiftMethod(prompt : JSON) -> String{

    let functionName = prompt["functionName"].string!
    let param = prompt["data"]

    var returnValue = "returnvalue"

    AppConstants.log("jsi_", "functionName: \(functionName) param: \(param)")

    switch functionName {
    case "functionOne":
        returnValue = handleFunctionOne(param: param)
    case "functionTwo":
        returnValue = handleFunctionTwo(param: param)
    default:
        returnValue = "returnvalue";
    }
    return returnValue
}
```

https://stackoverflow.com/questions/29249132/wkwebview-complex-communication-between-javascript-native-code/49474323