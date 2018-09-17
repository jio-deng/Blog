### Hybrid App背景

App开发工程师实在Google推出了Android、Apple推出iOS后出现的职位。当H5广泛流行之后（16年后Android 4.0以上的市场占有率已经超过70%，对H5的支持已经普及），针对App开发更加有效率的Hybrid方式开始流行。Android中有使用Chrome内核的Webview控件，可以使用H5来编写页面的主要逻辑，原生Webview用于加载显示，提高了开发效率。

目前App的开发模式更多还是原生开发和h5开发共存。也就是说，对于一些性能要求很高的页面模块，用原生来完成；对于一些通用型模块，用h5和js来完成。这种模式一般是一些有技术支持的公司自己实现的，包括H5和原生的通信：原生API提供，容器的一些处理全部由原生人员来完成。

---

### Android与H5的基本通信机制

**Native调用js**

4.4版本之前

```
// mWebView = new WebView(this); //即当前webview对象			
mWebView.loadUrl("javascript: 方法名('参数,需要转为字符串')"); 

//ui线程中运行
 runOnUiThread(new Runnable() {  
        @Override  
        public void run() {  
            mWebView.loadUrl("javascript: 方法名('参数,需要转为字符串')");  
            Toast.makeText(Activity名.this, "调用方法...", Toast.LENGTH_SHORT).show();  
        }  
});  
```
			
4.4以后(包括4.4)

```
//异步执行JS代码,并获取返回值	
mWebView.evaluateJavascript("javascript: 方法名('参数,需要转为字符串')", new ValueCallback() {
        @Override
        public void onReceiveValue(String value) {
    		//这里的value即为对应JS方法的返回值
        }
});
```

**Notice：**

1.Android 4.4之前使用loadUrl的方法来调用js方法，该方法只能调用js方法，但是无法获得返回值；而且该方法是在UI线程中被调用，可能会阻塞主线程

2.Android 4.4以上可以通过evaluateJavascript方法调用js方法，并且可以通过回调获得js方法的返回值；该方法是异步调用的，不会阻塞主线程

3.这种交互方式适用于少量数据的传输，如果是大量数据传输建议使用接口

**js调用Native**

Android中允许js调用的设置

```
WebSettings webSettings = mWebView.getSettings();  
 //Android容器允许JS脚本，必须要
webSettings.setJavaScriptEnabled(true);
//Android容器设置桥连对象
mWebView.addJavascriptInterface(getJSBridge(), "JSBridge");

private Object getJSBridge(){  
    Object insertObj = new Object(){  
    	@JavascriptInterface
        public String foo(){  
            return "foo";  
        }  
        
        @JavascriptInterface
        public String foo2(final String param){  
            return "foo2:" + param;  
        }  
          
    };  
    return insertObj; 
```
		
Html中JS调用原生的代码

```
//调用方法一
window.JSBridge.foo(); //返回:'foo'
//调用方法二
window.JSBridge.foo2('test');//返回:'foo2:test'
```
**Notice：**

Android 4.2之前，使用addJavascriptInterface方法为webview控件设置Js对象时，会被恶意软件通过反编译获取到Native注册的Js对象，然后通过反射Java内置静态类获取敏感信息等。
Android 4.2以后，需要在暴露的方法上添加@JavascriptInterface注释才可以进行调用，否则会找不到方法。JS能调用到已经暴露的api，并且能得到相应返回值。

---

### JsBridge

JsBridge定义了Native和H5交互的桥对象，互相之间的调用只通过这一个固定的桥对象。

这里的互相调用采用了触发url scheme的方式：

1.集成了JsBridge之后，使用BridgeWebView代替WebView；而在BridgeWebView的init中，会设置 `this.setWebViewClient(new BridgeWebViewClient(this));`

2.在BridgeWebViewClient中重载了以下方法，在页面加载结束时，将本地的js文件注入到页面中（文章下方对js文件解析）；拦截到特定url时进行交互处理

```
	@Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);

		//页面加载结束后，将本地js文件注入到页面中
        if (BridgeWebView.toLoadJs != null) {
            BridgeUtil.webViewLoadLocalJs(view, BridgeWebView.toLoadJs);
        }

        //startupMessage为消息队列，这里是将之前没有分发的消息进行分发并重置消息队列
        if (webView.getStartupMessage() != null) {
            for (Message m : webView.getStartupMessage()) {
                webView.dispatchMessage(m);
            }
            webView.setStartupMessage(null);
        }
    }
    
	@Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
			//解析拦截到的url
            url = URLDecoder.decode(url, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }

		//H5调用本地功能时，Js将消息内容放在sendMessageQueue中，并设置iframe的src为yy://__QUEUE_MESSAGE__/
        if (url.startsWith(BridgeUtil.YY_RETURN_DATA)) { // 如果是返回数据
            webView.handlerReturnData(url);
            return true;
        } else if (url.startsWith(BridgeUtil.YY_OVERRIDE_SCHEMA)) { //刷新消息队列
            webView.flushMessageQueue();
            return true;
        } else {
            return super.shouldOverrideUrlLoading(view, url);
        }
    }

```

如下图所示

![这里写图片描述](https://img-blog.csdn.net/20180916192536106?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzZDUwMTgyMzIwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

3 .	BridgeWebView中对不同的返回格式进行处理

```
	void flushMessageQueue() {
		if (Thread.currentThread() == Looper.getMainLooper().getThread()) {
			loadUrl(BridgeUtil.JS_FETCH_QUEUE_FROM_JAVA, new CallBackFunction() {
			//调用：javascript:WebViewJavascriptBridge._fetchQueue()
			//回调：判断Callback（Map中保存）中是否有对应回调：若有，则调用回调并移除；
			//若没有，则判断callbackId（由BridgeWebView调用doSend时生成，格式为`JAVA_CB_uniqueId_time`）。若有，则通过callbackId生成回调并重新发送
			//通过handlerName获取BridgeHandler并派发消息
			});
		}
	}
	/**
     * 获取到CallBackFunction data执行调用并且从数据集移除
     * @param url
     */
	void handlerReturnData(String url) {
		//获取函数名
		String functionName = BridgeUtil.getFunctionFromReturnUrl(url);
		//获取回调函数
		CallBackFunction f = responseCallbacks.get(functionName);
		//获取data
		String data = BridgeUtil.getDataFromReturnUrl(url);
		if (f != null) {
			f.onCallBack(data);
			responseCallbacks.remove(functionName);
			return;
		}
	}
```

* Native调用JS：

通过加载以javascript:开头的url即可实现调用Js的方法。

* JS调用Native：

向body中添加一个不可见的iframe元素。通过拦截url的方法来执行相应的操作，但是页面本身不能跳转，所以改变一个不可见的iframe的src就可以让webview拦截到url，而用户是无感知的。

iframe.src：

![这里写图片描述](https://img-blog.csdn.net/20180916180228232?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzZDUwMTgyMzIwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

---

### WebViewJavascriptBridge.js

本地js被注入到各个页面的js文件；提供初始化，注册Handler，调用Handler等方法。

```
	//sendMessage add message, 触发native处理 sendMessage
    function _doSend(message, responseCallback) {
        if (responseCallback) {
	        //与native相同的方式生成callbackId
            var callbackId = 'cb_' + (uniqueId++) + '_' + new Date().getTime();
            responseCallbacks[callbackId] = responseCallback;
            message.callbackId = callbackId;
        }

        sendMessageQueue.push(message);
        messagingIframe.src = 'yy://__QUEUE_MESSAGE__/'；
    }

    // 提供给native调用,该函数作用:获取sendMessageQueue返回给native,由于android不能直接获取返回的内容,所以使用url shouldOverrideUrlLoading 的方式返回内容
    function _fetchQueue() {
        var messageQueueString = JSON.stringify(sendMessageQueue);
        sendMessageQueue = [];
        //android can't read directly the return data, so we can reload iframe src to communicate with java
        bizMessagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://return/_fetchQueue/' + encodeURIComponent(messageQueueString);
    }

    //提供给native调用,receiveMessageQueue 在会在页面加载完后赋值为null,所以
    function _handleMessageFromNative(messageJSON) {
        console.log(messageJSON);
        if (receiveMessageQueue) {
            receiveMessageQueue.push(messageJSON);
        }
        _dispatchMessageFromNative(messageJSON);
    }
    
    //提供给native使用,
    function _dispatchMessageFromNative(messageJSON) {
        //解析json，获取responseId，通过id获取responseCallback
        //与BridgeWebView#flushMessageQueue()功能相同
        });
    }
```

---

### JsBridge流程图

最后贴上[撒网要见鱼](https://www.cnblogs.com/dailc/p/5931324.html)大佬的图：

![这里写图片描述](https://img-blog.csdn.net/20180916192724239?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzZDUwMTgyMzIwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

---

### 参考资料

[1.撒网要见鱼-Hybrid App基础篇系列文章](https://www.cnblogs.com/dailc/p/5931324.html)

[2.seph_von-JsBridge使用和原理](https://www.jianshu.com/p/910e058a1d63)
