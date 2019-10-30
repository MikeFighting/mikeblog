---
title: WebViewJavaScriptBridge源码剖析
date: 2017-05-26 16:15:00
tags: Objective-C
---

WebViewJavaScriptBridge是IOS中JS和OC交互的常用框架，它利用block的形式处理回调([相关Demo已上传](https://github.com/MikeFighting/OCJSBridgeDemo))，支持以下两种调用:

## 基本用法

它的两种使用场景如下：

![WebViewJavaScriptBridge使用场景](http://upload-images.jianshu.io/upload_images/1513759-77f94fd61ed13eef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### OC端的方法如下

  ![Method Frome OC](http://upload-images.jianshu.io/upload_images/1513759-d5f7a8ecdd8956de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  Method 1 是注册一个OC的方法--`testObjcCallback`,handler是JS掉用的内容，`responseCallback`是将OC处理返回给JS的回调(对应的是上述第2种调用)；
  Method 2 是调用JS的方法的`testJavascriptHandler`方法，`@{ @"foo":@"before ready" }`是需要传递的参数，`responseCallback`是将JS处理结果返回给OC的回调（对应的是上述的第1种调用）

### JS端的方法如下

  ![Method Frome JS](http://upload-images.jianshu.io/upload_images/1513759-eba410dda0ca2f3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  Method 1 是JS注册一个方法供OC调用，`responseCallback(responseData)`是将处理结果返回OC。
  Method 2 是在点击了一个按钮之后JS调用OC的方法，`{'foo': 'bar'}`是给OC的参数，response是OC处理后返回给JS的数据。
  *注：JS中是可以不写`;`号的，这和swift一样*
  
>JS调用OC，OC将处理结果回调给JS：要想被JS调用，我们首先要注册一个handler，和回调的            block，注册时候以键值对的形式存储这个block，handler，当JS调用OC时调用`webView:shouldStartLoadWithRequest:navigationType:`这个方法，根据JS传来的数据，找到之前保存的Block并且调用，同时新建一个需要把处理结果回调给JS的Blcok，OC处理完结果之后调用刚才创建的Block利用`stringByEvaluatingJavaScriptFromString`将处理结果返回给JS。
>OC调用JS时与此类似。基于这个流程，我们来看`WebViewJavaScriptBridge`的实现过程。

## 原理

接下来我们来分析从页面加载到OC和JS互相调用的整个过程：
  
### 准备工作

  当加载HTML文件的时候调用`[webView loadHTMLString:appHtml baseURL:baseURL];`,这时会调用：

```objc  
   - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
     if (webView != _webView) { return YES; }

     NSURL *url = [request URL];
     __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
     if ([_base isWebViewJavascriptBridgeURL:url]) {
         if ([_base isBridgeLoadedURL:url]) {
             [_base injectJavascriptFile];
         } else if ([_base isQueueMessageURL:url]) {
             NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
             [_base flushMessageQueue:messageQueueString];
         } else {
             [_base logUnkownMessage:url];
         }
         return NO;
     } else if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:shouldStartLoadWithRequest:navigationType:)]) {
         return [strongDelegate webView:webView shouldStartLoadWithRequest:request navigationType:navigationType];
     } else {
         return YES;
     }
 }
```

 在这个方法中判断URL的类型，如果是`WebViewJavascriptBridgeURL`那么就会判断是`BridgeLoadedURL`，`QueueMessageURL`还是未知的URL，在首次调用时是返回YES的，然后的URL就是`BridgeLoadedURL`，我们在看它的判断条件`[self isSchemeMatch:url] && [host isEqualToString:kBridgeLoaded];`Scheme是自己设置的`https`，那么`BridgeLoaded（__bridge_loaded__）`是什么呢？我们看`ExampleApp.html`文件，发现它的`script`标签中有这么一段代码:

```objc
    function setupWebViewJavascriptBridge(callback) {
        if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
        if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
        window.WVJBCallbacks = [callback];
        var WVJBIframe = document.createElement('iframe');
        WVJBIframe.style.display = 'none';
        WVJBIframe.src = 'https://__bridge_loaded__';
        document.documentElement.appendChild(WVJBIframe);
        setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
    }
  setupWebViewJavascriptBridge(function(bridge) {
   var uniqueId = 1
   function log(message, data) {
   var log = document.getElementById('log')
   var el = document.createElement('div')
   el.className = 'logLine'
   el.innerHTML = uniqueId++ + '. ' + message + ':<br/>' + JSON.stringify(data)
   if (log.children.length) { log.insertBefore(el, log.children[0]) }
   else { log.appendChild(el) }
  }
```

在这里我们发现了`https://__bridge_loaded__`这个iframe的`src`，并且在接下来调用`setupWebViewJavascriptBridge`时这个`src`会当做一个请求，这时会调用`shouldStartLoadWithRequest`这个方法。此时就满足了`isBridgeLoadedURL`这个请求。这时就会调用

```objc
    [_base injectJavascriptFile]
```

注入一个JS文件，这个JS文件的主要内容是（篇幅问题，有删减）:

```objc
 window.WebViewJavascriptBridge = {
  registerHandler: registerHandler,
  callHandler: callHandler,
  disableJavscriptAlertBoxSafetyTimeout: disableJavscriptAlertBoxSafetyTimeout,
  _fetchQueue: _fetchQueue,
  _handleMessageFromObjC: _handleMessageFromObjC
 };

 var messagingIframe;
 var sendMessageQueue = [];
 var messageHandlers = {};
 var CUSTOM_PROTOCOL_SCHEME = 'https';
 var QUEUE_HAS_MESSAGE = '__wvjb_queue_message__';
 var responseCallbacks = {};
 var uniqueId = 1;
 var dispatchMessagesWithTimeoutSafety = true;

 function registerHandler(handlerName, handler) {
  messageHandlers[handlerName] = handler;
 }
 function callHandler(handlerName, data, responseCallback) {
  _doSend();
 }
 function disableJavscriptAlertBoxSafetyTimeout() {
  dispatchMessagesWithTimeoutSafety = false;
 }
 function _doSend(message, responseCallback) {
  sendMessageQueue.push(message);
  messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
 }

 function _fetchQueue() {
  var messageQueueString = JSON.stringify(sendMessageQueue);
  sendMessageQueue = [];
  return messageQueueString;
 }
 function _dispatchMessageFromObjC(messageJSON) {
  if (dispatchMessagesWithTimeoutSafety) {
   setTimeout(_doDispatchMessageFromObjC);
  } else {
    _doDispatchMessageFromObjC();
  }
  
 function _doDispatchMessageFromObjC() {

   }
  }
 }

 function _handleMessageFromObjC(messageJSON) {
        _dispatchMessageFromObjC(messageJSON);
 }

 messagingIframe = document.createElement('iframe');
 messagingIframe.style.display = 'none';
 messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
 document.documentElement.appendChild(messagingIframe);

 registerHandler("_disableJavascriptAlertBoxSafetyTimeout", disableJavscriptAlertBoxSafetyTimeout);
 setTimeout(_callWVJBCallbacks, 0);
 function _callWVJBCallbacks() {
  var callbacks = window.WVJBCallbacks;
  delete window.WVJBCallbacks;
  for (var i=0; i<callbacks.length; i++) {
   callbacks[i](WebViewJavascriptBridge);
  }
 }
```

下面我们来分析下注入的JavaScript的内容。

1. 给window对象添加一个属性`WebViewJavascriptBridge`（*JS中可以直接给对象添加属性*）,这个对象包含以下内容：

1) registerHandler:注册调用方法
2）callHandler:调用OC时的方法
3）disableJavscriptAlertBoxSafetyTimeout:超时时弹框是否展示的标示
4）_fetchQueue:获取Queue对象的方法
5）_handleMessageFromObjC:处理OC调用的方法

2. 定义了一系列的变量来存储数据

messagingIframe:iframe标签，当我们的WebView加载它的时候，会调用其中的`src`,`src`就是调用请求的URL。
  
   1）sendMessageQueue:message数组
   2）messageHandlers:handler对象 *JS中{}表示对象*
   3）CUSTOM_PROTOCOL_SCHEME:scheme标示
   4）QUEUE_HAS_MESSAGE:有Message标识
   5）responseCallbacks:回调对象
   6）uniqueId:唯一标示ID
  
进过系列一的剖析，我们明白了使用`WebViewJavaScriptBridge`前需要做的准备工作，那么接下来，我们一起探讨`OC`和`JS`相互调用的具体执行过程以及其中的要点。

## JS调用OC，然后OC将处理结果返回JS

### OC首先注册JS将调用的方法

OC调用`registerHandler:`，这时将其调用信息存储在messageHandlers字典中以`handlerName`为Key,`给JS处理结果的Block`为Value(`_base.messageHandlers[handlerName] = [handler copy]`);

### 在JS中调用被注册的方法

JS调用

```js
    bridge.callHandler('testObjcCallback', {'foo': 'bar'}, function(response) {
    log('JS got response', response)
   })
```

来调用上文OC注册的方法，这个brige就是上文注入JS代码时候创建的，我们再它内部做了什么。

```js
    function callHandler(handlerName, data, responseCallback) {
        if (arguments.length == 2 && typeof data == 'function') {
            responseCallback = data;
            data = null;
        }
        _doSend({ handlerName:handlerName, data:data }, responseCallback);
    }
```

 这里判断了参数类型，如果传入的参数只有两个,并且第二个是`function`类型，那么就将第二个参数变为callBack,data置空，将handlerName和data转化成一个对象的两个属性并传给`_doSend()`。

```js
   function _doSend(message, responseCallback) {
          if (responseCallback) {
              var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
              responseCallbacks[callbackId] = responseCallback;
              message['callbackId'] = callbackId;
          }
          sendMessageQueue.push(message);
          messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
      }
```

这里的responseCallback是**JS先调用OC然后OC调用JS**时才会有的，如果这种情况，那么需要用唯一的标识（callbackId），来将这个responseCallback存储在`responseCallbacks`中，并且给message添加`callbackId` 这个属性。**这个数值会在下次OC调用JS的时候作为唯一的Key被用到。**软后将`message`放入:`sendMessageQueue`队列中，然后拼接src。

### 在回掉方法中拦截相应的方法，然后调用block

经过方法步骤2，会调用下面的回调方法

```objc
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType{}
```

在这个方法中调用

```objc
NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
[_base flushMessageQueue:messageQueueString];
```

 首先获取JS中的`messageQueue`(步骤2中的sendMessageQueue)，然后调用`flushMessageQueue：`方法：

```objc
    id messages = [self _deserializeMessageJSON:messageQueueString];
    for (WVJBMessage* message in messages) {
        if (![message isKindOfClass:[WVJBMessage class]]) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
            continue;
        }
        [self _log:@"RCVD" json:message];

        /////////*********OC先调用了JS,JS再调用了OC*********///////////
        NSString* responseId = message[@"responseId"];
        if (responseId) {
            //调用之前存储的Bolck
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];

        /////////*********JS先调用OC,OC再调用JS*********///////////
        /// 这里是JS先调用OC的时候存储的是 JS的回调函数
        } else {

            // JS先调用的OC,OC再调用JS
            WVJBResponseCallback responseCallback = NULL;
            NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    //JS调用OC时候的存储（后续OC调用JS返回计算结果）
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }

            WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
            if (!handler) {
                NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                continue;
            }
            //调用OC的Block,同时，如果OC调用responseCallback，则调用_queueMessage进行相应的处理
            handler(message[@"data"], responseCallback);
        }
    }
```

这里先将返回的JSON字符串转换成对象，这里的字符串是调用

```js
function _fetchQueue() {
  var messageQueueString = JSON.stringify(sendMessageQueue);
  sendMessageQueue = [];
  return messageQueueString;
    }
```

获取的，这里将`sendMessageQueue`转为JSON，然后将其置空，这里为啥使用数组而不用对象来存储呢？因为可能JS还没有处理结束就有两次调用，要保证他们不丢失使用了数组。然后判断数组中的Message对象是否有`responseId`(JS调用OC第一次时存储的),这里没有`responseId`所以走`else`:如果有`callbackId`(在JS中作为回调用的)，定义`responseCallback`，这个`block`就是OC将处理结果返回给JS时用到的block。如果没有`callbackId`说明，不需要回调JS，这个时候`responseCallback`为空。最后调用步骤1中存储在`messageHandlers`对象中的block，并且将刚才创建的`responseCallback`作为参数传入，以便OC将计算结果传递给JS。

### OC将计算结果返回给JS

```objc
    [_bridge registerHandler:@"testObjcCallback" handler:^(id data, WVJBResponseCallback responseCallback) {

        NSLog(@"testObjcCallback called: %@", data);
        responseCallback(@"response form oc's call back");  
    }];
```

在`handler`的最后一步调用`responseCallback()`将处理结果回调给JS。这个`responseCallback()`就是我们在步骤3中创建的`responseCallback`。我们再来看这个block。看步骤3可以看到这个其内部调用

      [self _queueMessage:msg];
      [self _dispatchMessage:message];

在`_dispatchMessage`内部执行:

```objc
     NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
```

接下来JS中的`_handleMessageFromObjC`就会接收到OC传过来处理结果。

```js
     function _doDispatchMessageFromObjC() {
            var message = JSON.parse(messageJSON);
            var messageHandler;
            var responseCallback;

            if (message.responseId) {
                responseCallback = responseCallbacks[message.responseId];
                if (!responseCallback) {
                    return;
                }
                responseCallback(message.responseData);
                delete responseCallbacks[message.responseId];
            } else {
               // OC先调用JS是用到
            }
        }
```

这个时候我们看到了步骤三中的`responseId`的作用了，这时候`responseId`就表明了是OC将处理结果传递给JS并不需要JS再调用OC了，这时只调用`responseCallback(message.responseData);`将数据传给JS。
这样我们就完成了JS调用OC，然后OC将结果回调给JS的全部过程。

## OC调用JS,然后JS将处理结果返回给OC

### JS注册相应的方法供回调

   同OC注册方法时候一样，JS也是用一个`messageHandlers`对象来存储

```js
        function registerHandler(handlerName, handler) {
        messageHandlers[handlerName] = handler;
        }
```

### OC调用JS时存储调用信息

```objc
- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
      NSMutableDictionary* message = [NSMutableDictionary dictionary];

      if (data) {
          message[@"data"] = data;
      }

      if (responseCallback) {
          NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
          self.responseCallbacks[callbackId] = [responseCallback copy];
          message[@"callbackId"] = callbackId;
      }

      if (handlerName) {
          message[@"handlerName"] = handlerName;
      }
      [self _queueMessage:message];
  }
```

这里使用`message`字典来存储参数，方法名，使用`responseCallbacks`来存储JS处理完之后，需要回调的Block(这里为了确保多次调用不会覆盖之前的调用，使用了唯一的callbackId)。
同上文所述，最终会调用

```objc
- (NSString*) _evaluateJavascript:(NSString*)javascriptCommand {
return [_webView stringByEvaluatingJavaScriptFromString:javascriptCommand];
}
```

### JS调用`_dispatchMessageFromObjC`

   这时message没有`responseId`，会走`else`，

```js
if (message.callbackId) {
                var callbackResponseId = message.callbackId;
                responseCallback = function(responseData) {
                    _doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
                };
            }
            var handler = messageHandlers[message.handlerName];
            if (!handler) {
                console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
            } else {
                handler(message.data, responseCallback);
            }
```

这里定义了需要给OC传递结果的`responseCallback`，取出之前注册的`handler`:`messageHandlers[message.handlerName]`，然后调用这个`handler`，并将这个`responseCallback`作为参数传进去,`handler(message.data, responseCallback);`

### JS将结果回传给OC

在步骤三中调用handler:

```js
    function(data, responseCallback) {
    log('ObjC called testJavascriptHandler with', data)
    var responseData = { 'Javascript Says':'Right back atcha!' }
    log('JS responding with', responseData)
    responseCallback(responseData)
   }
```

在这个`handler`的结尾调用步骤三种的`responseCallback`（传入的只有数据没有回调），根据步骤三可以看出来其会调用`_doSend`方法。该方法中由于没有传进去回调，所以不会给message对象添加`callbackId`，只调用：

```js
sendMessageQueue.push(message);
messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
```

这是由于含有`responseId`(在步骤三中的`_doSend`调用时设置)，所以只会取出之前存储的block,并且将结果回传给OC:

```objc
//调用之前存储的Bolck
WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
responseCallback(message[@"responseData"]);
[self.responseCallbacks removeObjectForKey:responseId];
```

 至此，OC和JS交互的所有逻辑已介绍完毕（WKWebView实现方式相同），总结下两种情景的回调，其实现方式及其相似，正如文章开头的总结。