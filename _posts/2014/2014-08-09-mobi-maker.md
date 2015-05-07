---
layout: post
title: mobi文件的一种制作方法
categories:
- Other
tags:
- kindle
- mobi
- nodejs
---

##背景
今年上半年，入了一个kindle paperwhite来看看电子书，挺好用的。但是在使用中发现，平时除了看一些电子书之外，也会看网页文章。更多的时候是在上下班路途中，地铁上无3g信号等恶劣环境下想看看书。所以，如果能把平时来不及看的文章转到kindle上，将会让上班路上的旅程非常愉快。:D

##解决方案
先写方案，可以简单归纳为以下几点：

* 定义配置。声明mobi文件的标题、作者、文章链接
* 根据文章链接，抓取文章内容（html格式）
* 组合内容，写入一个html文件
* 使用工具kindlegen，处理html文件，最终生成mobi文件
* 拿到mobi文件之后，通过邮件的方式传到kindle即可

##具体实现
###**定义配置**
配置文件代码片段如下：
{% highlight javascript %}
module.exports = {
  dir : '20140706/',//生成mobi文件的目录
  title : 'mobi_demo',//mobi文件标题
  author : 'aslinwang',//mobi文件作者
  urls : [//文章链接数组
    'http://greengerong.github.io/blog/2013/04/14/li-yong-Travis-CI-rang-ni-di-github-xiang-mu-chi-xu-gou-jian-Node-js-wei-li/',
    'http://blog.jobbole.com/63770/?from=timeline&isappinstalled=0',
    'http://mp.weixin.qq.com/s?__biz=MjM5NjY5NTM0MQ==&mid=201123995&idx=1&sn=70484e70a1f6fe63f7ef86b72f358cc0&scene=2&from=timeline&isappinstalled=0#rd',
    'http://www.oschina.net/translate/mistakes-avoid-responsive-web-design',
    'http://jser.it/blog/2014/07/07/numbers-in-javascript/'
  ]
};
{% endhighlight %}

###**抓取网页内容**
这里使用了readability.com的一个服务，调用其[API](readability.com/api/content/v1/parser?token=abcdefg&url=http://baidu.com)并传入文章链接，可以获取到文章html内容。
API中**token**参数需要到其[网站](https://www.readability.com/settings/account)注册申请，**url**参数为文章链接
代码片段如下：
{% highlight javascript %}
var parse = (function(){
  var jsons = [];
  var defer = q.defer();

  var action = function(urls){
    var url = urls.shift();
    if(!url){
      defer.resolve(jsons);
      jsons = [];
      return
    }
    console.log(url);
    var data = {
      token : config.TOKEN,
      url : url
    }
    var req = https.request({
      hostname : 'readability.com',
      port : 443,
      method : 'GET',
      path : '/api/content/v1/parser?' + qs.stringify(data)
    }, function(res){
      var _chunk;
      res.setEncoding('utf-8');
      res.on('data', function(chunk){
        _chunk = _chunk ? _chunk + chunk : chunk;
      });
      res.on('end', function(){
        var json = JSON.parse(_chunk);
        jsons.push(json);

        action(urls);
      });
    });

    req.on('error', function(e){
      defer.reject('parse request error!');
    });

    req.end();

    return defer.promise;
  }

  return action;
}());
{% endhighlight %}

###**组合内容为html文件**
上面的步骤可以获取到各个文章的html内容，这个步骤就是简单的将这些内容组合成一个html文件。代码片段如下所示：
{% highlight javascript %}
var makeMobi = function(info){
  DATA_PATH = DATA_PATH + info.dir;
  //make html 文件操作
  var dest = './modules/mobi/data/' + info.dir + info.title + '.html';
  var html = [
    '<html>',
      '<head>',
        '<title><%=title%></title>',
      '</head>',
      '<body>',
      '<%for(var i=0;i<pages.length;i++){ %>',
        '<h1><%=pages[i].title%></h1>',
        '<%=pages[i].content%>',
        '<br /><br />',
      '<% } %>',
      '</body>',
    '</html>'
  ].join('');
  var medias = [];
  for(var i = 0; i < info.pages.length; i++){
    info.pages[i].title = util.encodeGB2312(info.pages[i].title);

    info.pages[i].content = parseMedia(info.pages[i].content);
  }
  info.title = util.encodeGB2312(info.title);
  html = util.txTpl(html, info);
    html = cureHtml(html);
  fs.writeFile(dest, html, function(e){
    fetchMedia().done(function(){
      //html -> mobi
      cp.exec('kindlegen ' + dest, function(err, stdout, stderr){
        console.log('kindlegen log>>>', stdout);
        console.log('kindlegen err>>>', stderr);
      });
    });
  });
};
{% endhighlight %}
需要注意的是，需要对内容中所含的中文进行编码，否则在kindle中阅读的时候，中文将会是乱码。
还有一点是上面的代码对图片(<img>)的处理。程序将图片从网络上拉取存在本地，然后将图片的src改成本地地址。这么做是因为在下面生成mobi文件的时候，kindlegen不能拉取网络图。

###**生成mobi文件**
在上面的代码中，其实已经涉及到了生成mobi文件的过程，简单执行一个命令行即可。具体kindlegen的用法可以通过help了解
{% highlight c# %}
cp.exec('kindlegen ' + dest, function(err, stdout, stderr){
    console.log('kindlegen log>>>', stdout);
    console.log('kindlegen err>>>', stderr);
});
{% endhighlight %}``
拿到mobi文件之后，后面的过程就是上传到kindle中了。

##使用方法

* 在modules/mobi/data目录下新建一个格式为"yyyymmdd"的目录，并编写配置文件index.js
* 输入命令：
{% highlight bash %}
node app.js -mobi 20140809
{% endhighlight %}

##参考
* https://www.readability.com/developers/api/parser (readability文档)
* http://www.idpf.org/epub/20/spec/OPF_2.0.1_draft.htm (OPF格式规范)
* http://www.amazon.com/gp/feature.html?ie=UTF8&docId=1000765211 (kindlegen下载地址，注册账户所在国籍必须是美帝才能下载。。)
* http://aslinwang.u.qiniudn.com/Android_Training.mobi (利用此方法制作的Android官网Training文件)

##项目主页
https://github.com/aslinwang/aslin

https://github.com/aslinwang/aslin/modules/mobi (mobi制作模块)
