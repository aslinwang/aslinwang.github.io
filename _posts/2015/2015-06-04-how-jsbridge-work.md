---
layout: post
title: TGJSBridge工作原理
categories:
- 前端
tags:
- native通信
- jsbridge
---

在iOS中，TGJSBridge框架提供了一种obj-c与js通信的方案。使用TGJSBridge之后，js可以方便的调用native提供的原生方法来实现相应的逻辑。

###TGJSBridge的使用
js端
{% highlight javascript %}
jsBridge.postNotification('setTitle', {
    title : data.value
});
{% endhighlight %}
obj-c端
{% highlight objective-c %}
- (void)webViewDidFinishLoad:(UIWebView *)webView
{
    [super webViewDidFinishLoad:webView];
    NSString *jsBridgePath = [[NSBundle mainBundle] pathForResource:@"TGJSBridge" ofType:@"js"];
    NSString *jsBridge = [NSString stringWithContentsOfFile:jsBridgePath encoding:NSUTF8StringEncoding error:nil];
    [self.webView stringByEvaluatingJavaScriptFromString:jsBridge];
    
    [self.webView stringByEvaluatingJavaScriptFromString:@"window.onJsBridgeReady()"];
    
    [self showLoading:NO animated:YES];
}

/**
 *  设置文章的标题
 */
- (void)jsBridge:(TGJSBridge *)bridge setTitleWithUserInfo:(NSDictionary *)userInfo fromWebView:(UIWebView *)webview
{
//    NSString *titleName = QNString([userInfo objectForKey:@"title"], _carSerialName);
    ((QNTitleView *)self.qn_navigationItem.titleView).text = QNString([userInfo objectForKey:@"title"], _carSerialName);
}
{% endhighlight %}
所以，obj-c端会先获取到TGJSBridge.js文件并执行，然后当js端发送“setTitle”通知之后，obj-c端会有相对应的方法来处理这个通知。但是，查看TGJSBridge的源码，并没有发现有定义包含“setTitleWithUserInfo”标记名的消息定义。所以为了了解通知具体是如何通过TGJSBridge从js端传递到obj-c端的，就需要看看TGJSBridge的实现原理。

###TGJSBridge的原理
TGJSBridge框架包括两部分

* TGJSBridge.js——定义JSBridge类，实现js与obj-c间互相发送通知、js绑定通知与解除绑定，以及js发送通知的具体方式
* TGJSBridge.h/TGJSBridge.m——接收来自js端的通知，解析通知参数，调用绑定这个通知的处理函数

js发送通知
{% highlight javascript %}
function bridgeCall(src,callback) {
    iframe = document.createElement("iframe");
    iframe.style.display = "none";
    iframe.src = src;
    var cleanFn = function(state){
       console.log(state) 
        try {
            iframe.parentNode.removeChild(iframe);
        } catch (error) {}
        if(callback) callback();
    };
    iframe.onload = cleanFn;
    document.documentElement.appendChild(iframe);
}

bridgeCall('jsbridge://PostNotificationWithId-' + this.notificationIdCount);
{% endhighlight %}
通知传递是通过伪协议的方式，html与native约定一个协议，当html以这个协议发送请求时，会被native拦截。

obj-c接收通知并处理
{% highlight objective-c %}
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    NSURL *url = [request URL];
    
    if ([[[url scheme] lowercaseString] isEqualToString:kTGJSBridgeProtocolScheme])
    {//协议验证通过，被拦截下来，不在向Internet转发
        [self dispatchNotification:[url host] fromWebView:webView];
        return NO;
    }
    else
    {
        //forward
        if (self.delegate != nil && [self.delegate respondsToSelector:@selector(webView:shouldStartLoadWithRequest:navigationType:)]) {
            return [self.delegate webView:webView shouldStartLoadWithRequest:request navigationType:navigationType];
        }
        return YES;
    }   
}

// 通知派发消息定义
- (void)dispatchNotification:(NSString*)notificationString 
                 fromWebView:(UIWebView*)webView
{
    if ([notificationString isEqualToString:kTGJSBridgeNotificationReady])
    {
        //push notificationQueue to webView
        NSMutableArray *notificationQueue = [self notificationQueueForWebView:webView];
        for(NSDictionary *notification in notificationQueue)
        {
            [self triggerJSEvent:[notification objectForKey:@"name"] userInfo:[notification objectForKey:@"userInfo"] toWebView:webView];
        }
    }
    else if([notificationString hasPrefix:kTGJSBridgePostNotificationWithId])
    {
        NSRange range = [notificationString rangeOfString:kTGJSBridgeNotificationSeparator];
        NSInteger index = range.location + range.length;
        NSString *notificationId = [notificationString substringFromIndex:index];
        //processNotification
        NSDictionary *responseDict = [self fetchNotificationWithId:notificationId fromWebView:webView];
        
        QN_D(@"dispatch notification %@", responseDict[@"name"]);
        
        if(self.delegate) {
            @try {
                [self.delegate jsBridge:self
            didReceivedNotificationName:[responseDict objectForKey:@"name"]
                               userInfo:[responseDict objectForKey:@"userInfo"]
                            fromWebView:webView];
            }
            @catch (NSException *exception) {
            }
        }
    }
    else
    {
        //processError
    }
}
{% endhighlight %}
在以上过程中，调用了一个“didReceivedNotificationName”方法。TGJSBridge中定义了一个protocol叫TGJSBridgeDelegate，didReceivedNotificationName是这个protocol的成员方法，需要具体的业务Controller来实现这个接口。
{% highlight objective-c %}
@interface CWebViewController : QNRootViewController<TGJSBridgeDelegate> {
    NSString *_requestUrl;
    NSString *_callbackUrl;
    BOOL _shouldRrefresh;
    CWebView *_webView;
    id _observer;
}

// 方案一
- (void)jsBridge:(TGJSBridge *)bridge
        didReceivedNotificationName:(NSString *)name
                           userInfo:(NSDictionary *)userInfo
                        fromWebView:(UIWebView *)webview {
    if ([name isEqualToString:@"post"]) {
        NSString *requestName = @"";
        if ([self.channel isEqualToString:kChannelIDCaiJing]) {
            requestName = kSpeedTestPluginStock;
        } else if ([self.channel isEqualToString:kChannelIDTiYu]) {
            requestName = kSpeedTestPluginSport;
        }
        [self.httpBridge post:userInfo withDelegate:webview requestName:requestName];
        return;
    } else if ([name isEqualToString:@"onWebCellReady"]) {
        [self performSelector:@selector(onWebCellReady) withObject:nil afterDelay:0.5f];
        return;
    } else if ([name isEqualToString:@"onWebCellError"]) {
        [self onWebCellError];
        return;
    } else if ([name isEqualToString:@"onWebCellUIChanged"]) {
        [self saveShortcut];
        return;
    }
}

// 方案二
- (void)jsBridge:(TGJSBridge *)bridge
        didReceivedNotificationName:(NSString *)name
                           userInfo:(NSDictionary *)userInfo
                        fromWebView:(UIWebView *)webview {
    if (![self do_jsBridge:bridge notification:name userInfo:userInfo webView:webview])
        QN_W(@"unknown notification: %@, userinfo %@", name, userInfo);
}
- (BOOL)do_jsBridge:(TGJSBridge *)bridge
        notification:(NSString *)name
            userInfo:(NSDictionary *)userInfo
             webView:(UIWebView *)webview {
    if ([name isEqualToString:kQNWebViewControllerJSNotificationGetInstallState]) {
        // do something
        return YES;
    } else {
        NSString *functionName = [NSString stringWithFormat:@"jsBridge:%@WithUserInfo:fromWebView:", name];
        SEL selector = NSSelectorFromString(functionName);
        if ([self respondsToSelector:selector]) {
            ((void (*)(id, SEL, id, id, id))objc_msgSend)(self, selector, bridge, userInfo, webview);
            return YES;
        } else if ([self.jsBridgeDelegate respondsToSelector:selector]) {
            ((void (*)(id, SEL, id, id, id))objc_msgSend)(self.jsBridgeDelegate, selector, bridge, userInfo, webview);
            return YES;
        } else {
            QNWebViewHelper *webViewHelper = [QNWebViewHelper sharedHelper];
            if ([webViewHelper respondsToSelector:selector]) {
                ((void (*)(id, SEL, id, id, id))objc_msgSend)(webViewHelper, selector, bridge, userInfo, webview);
                return YES;
            }
        }
    }
    return NO;
}
{% endhighlight %}
方案一这种消息实现方式，是直接获取通知名，然后做对应的处理。缺点是，必须在这个controller中来做派发，耦合比较深。

方案二利用obj-c动态绑定的特性，以函数指针的方式，获取"{通知名}WithUserInfo"格式的方法并执行。NSSelectorFromString可以在当前类包括子类查找指定的方法。这个方案可以将具体的逻辑放在子类中处理，在子类中定义"{通知名}WithUserInfo"格式的方法即可。