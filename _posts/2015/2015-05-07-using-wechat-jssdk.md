---
layout: post
title: 微信jssdk使用初探
categories:
- 前端
tags:
- 微信
- jssdk
---

##背景
微信在升级到6.0版本之后，引入了jssdk。
jssdk是微信公众平台面向网页开发者提供的基于微信内的网页开发工具包。通过jssdk，可以使用微信内的拍照、选图、语音、位置等手机系统的功能，还能使用微信分享、扫一扫、卡券、支付等功能。

##问题
jssdk的引入产生的一个问题就是，以前在页面中只要放入WeixinJSBridge，就能使用的分享功能，目前在新版本的微信就不好使了。
{% highlight javascript %}
function onBridgeReady(){
   //转发朋友圈
  WeixinJSBridge.on("menu:share:timeline", function(e) {
    var data = {
      img_url: ShareCfg.img,
      img_width: ShareCfg.img_w,
      img_height: ShareCfg.img_h,
      link: ShareCfg.url,
      desc:ShareCfg.desc,
      title: ShareCfg.title
    };
    WeixinJSBridge.invoke("shareTimeline", data, function(res) {
      WeixinJSBridge.log(res.err_msg)
    });
  });

  //分享给朋友
  WeixinJSBridge.on('menu:share:appmessage', function(argv) {
    WeixinJSBridge.invoke("sendAppMessage", {
      img_url: ShareCfg.img,
      img_width: ShareCfg.img_w,
      img_height: ShareCfg.img_h,
      link: ShareCfg.url,
      desc: ShareCfg.desc,
      title: ShareCfg.title
    }, function(res) {
      WeixinJSBridge.log(res.err_msg)
    });
  });
}

try{
  document.addEventListener('WeixinJSBridgeReady', function(){
    onBridgeReady();
  })
}catch(e){

}
{% endhighlight %}
相同的代码在6.0及以上版本的微信中，并不能改变分享的缩略图和文案。是因为微信已经禁用这种方式。继续使用分享需要用jssdk的方式。
{% highlight javascript %}
//分享到朋友圈
wx.onMenuShareTimeline({
  title: '', // 分享标题
  link: '', // 分享链接
  imgUrl: '', // 分享图标
  success: function () { 
    // 用户确认分享后执行的回调函数
  },
  cancel: function () { 
    // 用户取消分享后执行的回调函数
  }
});
//分享给好友
wx.onMenuShareAppMessage({
  title: '', // 分享标题
  desc: '', // 分享描述
  link: '', // 分享链接
  imgUrl: '', // 分享图标
  type: '', // 分享类型,music、video或link，不填默认为link
  dataUrl: '', // 如果type是music或video，则要提供数据链接，默认为空
  success: function () { 
    // 用户确认分享后执行的回调函数
  },
  cancel: function () { 
    // 用户取消分享后执行的回调函数
  }
});
{% endhighlight %}
调用方式与之前相比，差异不大，不同的是引入了wx对象。这个对象在微信提供的[js文件](http://res.wx.qq.com/open/js/jweixin-1.0.0.js)中定义，使用之前需要配置。代码如下：
{% highlight javascript %}
wx.config({
  debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
  appId: '', // 必填，公众号的唯一标识
  timestamp: , // 必填，生成签名的时间戳
  nonceStr: '', // 必填，生成签名的随机串
  signature: '',// 必填，签名，见附录1
  jsApiList: [] // 必填，需要使用的JS接口列表，所有JS接口列表见附录2
});
{% endhighlight %}
这里面比较棘手的就是需要传appId,timestamp,nonceStr,signature。而这些参数是需要依赖后台才能获得的。所以以前简单的分享功能，只需要前端同学一人搞定，现在就不行了，需要后台同学的配合，根据微信的jssdk使用权限签名算法来获取参数timestamp,nonceStr,signature

##jssdk使用权限签名算法node版本
但是，在可以运行node的环境下，前端还是可以自己搞定使用权限的签名。（好吧，精通全栈的牛人表示不用node也能搞定）。在实现签名算法之前，根据微信文档先在微信公众平台绑定域名，引入js文件。具体步骤参考[微信官方文档](http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html)
{% highlight javascript %}
var getSign = function(url){
  var defer = q.defer();
  var deltaT = +new Date() - timer;

  async.waterfall([
    function gettoken(cb){// 获取access_token
      if(deltaT > EXPIRE || !sign){
        getToken(cb);
      }
      else{
        cb(null, cache.a_t);
      }
    },
    function getticket(a_t, cb){// 获取jsapi_ticket
      if(a_t){
        if(deltaT > EXPIRE || !sign){
          getTicket(a_t, cb);
        }
        else{
          cb(null, cache.tkt);
        }
      }
      else{
        cb(null);
      }
    },
    function getsign(tkt, cb){// 根据签名算法进行签名
      if(tkt){
        sign = {};
        sign.appId = APPID;// appid
        sign.nonceStr = 'aslinwang';// nonceStr，自己随便定义
        sign.timestamp = Math.round(+new Date() / 1000);// timestamp 时间戳
        
        var str = [
          'jsapi_ticket=', tkt,
          '&noncestr=', sign.nonceStr,
          '&timestamp=', sign.timestamp,
          '&url=', url
        ].join('');

        var sha1 = crypto.createHash('sha1');
        sha1.update(str);

        cb(null, sha1.digest('hex'));
      }
      else{
        cb(null);
      }
    }
  ], function(err, res){
    sign.signature = res;// signature 签名 
    defer.resolve(sign);
  });

  return defer.promise;
};

function getToken(cb){
  var opts = {
    hostname : 'api.weixin.qq.com',
    port : 443,
    path : ['/cgi-bin/token?grant_type=client_credential&appid=', APPID, '&secret=', APPSECRET].join(''),
    method : 'GET'
  };

  var req = https.request(opts, function(res){
    var _chunk;
    res.on('data', function(chunk){
      _chunk = _chunk ? _chunk + chunk : chunk;
    });

    res.on('end', function(){
      timer = +new Date();
      
      if(cb){
        _chunk = JSON.parse(_chunk);
        cache.a_t = _chunk.access_token;
        cb(null, _chunk.access_token);
      }
    });
  });

  req.end();

  req.on('error', function(e){
    conosole.log(e);
    cb(null, '');
  });
}

function getTicket(token, cb){
  var opts = {
    hostname : 'api.weixin.qq.com',
    port : 443,
    path : ['/cgi-bin/ticket/getticket?access_token=', token , '&type=jsapi'].join(''),
    method : 'GET'
  };

  var req = https.request(opts, function(res){
    var _chunk;
    res.on('data', function(chunk){
      _chunk = _chunk ? _chunk + chunk : chunk;
    });

    res.on('end', function(){
      _chunk = JSON.parse(_chunk);
      cache.tkt = _chunk.ticket;
      cb(null, _chunk.ticket);
    });
  });

  req.end();

  req.on('error', function(e){
    console.log(e);
    cb(null, '');
  });
}
{% endhighlight %}
拿到这几个参数之后，jssdk中的接口就可以随便使用了。

##注意事项

* access_token和jsapi_ticket有效期是7200s，开发者必须在自己的服务全局缓存这两个值
* 绑定域名的公众号必须要通过微信认证
* 以qq.com为根域名的页面，老的WeixinJSBridge方案仍然被兼容，但是建议都切到jssdk方案，以免随着后面微信版本的升级，不再兼容WeixinJSBridge方案
* 公司内部josephzhou哥将jssdk做了一次封装，实现了自动获取timestamp,nonceStr,signature参数的过程，试用之后感觉挺方便的。申请接入，[点此了解](http://km.oa.com/group/1663/articles/show/215876)（公司内网地址）

##参考
* http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html
