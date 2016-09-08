# iOS原生和Web交互方案探讨
----

## UIWebView 
### 描述
我从事iOS的开发比较早，那个时候iOS开发使用原生控件是主流。但有时需要加载网页，那么此时就需要到了UIWebView。随着iOS开发的深入，以及多平台的兼容，很多App多少需要使用到Web。
> 网上有现成的方案[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)

### 使用方式
UIWebView中原生和Web的交互，可以使用两个方法来进行:

- 原生调用Web
 	`- (NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;`

- Web调用原生: 苹果没有提供直接的方案，添加一个隐藏的iframe设置URL，然后在UIWebViewDelegate中进行拦截(Cordova就是使用这种机制)
	`- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;`

### 源代码
- [Demo地址] (https://github.com/yincheng1988/WebViewDemo)

## JavaScriptCore
### 描述
iOS7以后，苹果推出了JavaScriptCore框架。将WebKit中的JavaScript引擎进行了封装。这样我们就可以调用一个运行JavaScript代码的环境了。

### 使用方式
- 协同UIWebView使用，在WebView加载成功后，获取运行环境JSContext
	`JSContext *context = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];`
- 通过`- (JSValue *)evaluateScript:(NSString *)script;`执行JavaScript代码。这样就完成了原生调用Web了。
- Web调用原生，OC代码中需要实现JSExport协议
	```
	// OC中定义
	JSContext[@"consoleLog"] = ^(NSString *str) {
    NSLog(@"%@", str);
  	};
  	// JS调用
  	consoleLog('test');
	```
- JSValue中提供了Web和原生中的对象的转换表

	Objective-C type  |   JavaScript type
 	--------------------+---------------------
         nil         |     undefined
        NSNull       |        null
       NSString      |       string
       NSNumber      |   number, boolean
     NSDictionary    |   Object object
       NSArray       |    Array object
        NSDate       |     Date object
       NSBlock (1)   |   Function object (1)
          id (2)     |   Wrapper object (2)
        Class (3)    | Constructor object (3)
- Note：Web调用原生后，所执行的Block是在子线程中，所以UI的操作，需要放入主线程中

### 源代码
- [Demo地址] (https://github.com/yincheng1988/WebViewDemo)

### 参考资料
- [NSHipster](http://nshipster.cn/javascriptcore/)

## WKWebView
### 描述
iOS8以后，推出了新框架WebKit和WKWebView，用于替代老旧的UIKit中的UIWebView。
官方在说明中强调WKWebView的优点：
> 1. 性能和稳定性上的极大提升；
2. 60fps的滚动刷新率以及内置手势；
3. 高效的app和web信息交换通道；
4. 使用和Safari相同的JavaScript 引擎；
5. 将UIWebView和UIWebViewDelegate在WKWebKit中重构成14个类以及3个协议；

### 使用方式
- 页面加载

	```
	- (WKNavigation *)loadRequest:(NSURLRequest *)request;
	- (WKNavigation *)loadHTMLString:(NSString *)string baseURL:(NSURL *)baseURL;
	
	// iOS9新增
	- (WKNavigation *)loadFileURL:(NSURL *)URL allowingReadAccessToURL:(NSURL *)readAccessURL;
	- (WKNavigation *)loadData:(NSData *)data MIMEType:(NSString *)MIMEType characterEncodingName:(NSString *)characterEncodingName baseURL:(NSURL *)baseURL;
	
	// 示例
	[wkWebView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"https://baidu.com"]]];
	NSURL *URL = [NSURL fileURLWithPath:[[NSBundle mainBundle] pathForResource:@"index" ofType:@"html"]];
	[wkWebView loadFileURL:URL allowingReadAccessToURL:[NSBundle mainBundle].resourceURL];
	```

- 访问历史

	> WKWebView新增了访问指定历史记录，在UIWebView中也有方案实现。
  参考[IMYWebView.m](https://github.com/wangyangcc/IMYWebView/blob/master/Classes/IMYWebView.m)实现

	```
	- (WKNavigation *)goToBackForwardListItem:(WKBackForwardListItem *)item;

	// UIWebView
	[UIWebView stringByEvaluatingJavaScriptFromString:[NSString stringWithFormat:@"window.history.go(-%ld)", (long)index]];

	// WKWebView
	WKBackForwardListItem *item = WKWebView.backForwardList.backList[index];
	[WKWebView goToBackForwardListItem:item];
	```

- 调用JavaScript

	> UIWebView中调用JS的API是同步，而WKWebView中调用JS的API是异步。
	
	```
	// UIWeView
	- (NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;
	
	// WKWebView
	- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^)(id, NSError *error))completionHandler;
```

- JavaScript与原生的通信

	- WebKit中提供了`WKUserScript`类来进行JS脚本注入
	- 原生通过注册`WKScriptMessageHandler`协议，让JavaScript与原生进行通信：`window.webkit.messageHandlers.{name}.postMessage({body})`
	
	```
	// 注入JS脚本
	NSString *jsString = @"function message(msg) { alert(msg); }";
	WKUserScript *script = [[WKUserScript alloc] initWithSource: jsString injectionTime:injectionTime forMainFrameOnly:YES];
	[WKWebView.userContentController addUserScript:script];
	
	// 注册通信协议消息
	[WKWebView.userContentController addScriptMessageHandler:id<WKScriptMessageHandler>handler name:@"message"];
	// JS与原生进行通信
	window.webkit.messageHandlers. message.postMessage('Test');
	```

- `WKNavigationDelegate`协议
	
	```
	// 页面开始加载时调用
	WKWebView: - (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(WKNavigation *)navigation;
	UIWebView: - (void)webViewDidStartLoad:(UIWebView *)webView;

	// 当内容开始返回时调用
	WKWebView: - (void)webView:(WKWebView *)webView didCommitNavigation:(WKNavigation *)navigation;
	
	// 页面加载完成之后调用
	WKWebView: - (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation;
	UIWebView: - (void)webViewDidFinishLoad:(UIWebView *)webView;

	// 页面加载失败时调用
	WKWebView: - (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(WKNavigation *)navigation;
	UIWebView: - (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;
	
	/// 页面跳转
	WKWebView:
	// 接收到服务器跳转请求之后调用
	- (void)webView:(WKWebView *)webView didReceiveServerRedirectForProvisionalNavigation:(WKNavigation *)navigation;
	// 在收到响应后，决定是否跳转
	- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler;
	// 在发送请求之前，决定是否跳转
	- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler;
	
	UIWebView:
	- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
	```

- 加载进度`estimatedProgress`和标题`title`

	> 可以非常方便的通过KVO来进行监听

- `WKUIDelegate`协议

	> 通过此协议使用原生UI来定义Web中的Alert、Confirm、Prompt控件

- 注意事项

	- iOS9后提供`loadFileURL:allowingReadAccessToURL:`方式加载本地HTML。
	- 但是在iOS9之前使用`loadRequest:`方式加载本地HTML，会出现无法读取的问题。[解决方案](http://stackoverflow.com/questions/24882834/wkwebview-not-loading-local-files-under-ios-8 http://www.jianshu.com/p/ccb421c85b2e)

### 源代码
- [Demo地址] (https://github.com/yincheng1988/WebViewDemo)

### 参考资料
- [NSHipster](http://nshipster.cn/wkwebkit/)


### Cordova

### Crosswalk
