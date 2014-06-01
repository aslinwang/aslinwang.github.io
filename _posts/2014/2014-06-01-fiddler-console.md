---
layout: post
title: 一种移动端web开发调试方案
categories:
- Programming
tags:
- Fiddler
- mobile
- debug
---

##背景
在PC上做web前端开发的时候，通过浏览器的一些开发者工具，可以非常方便的调式和打印log。但是，如果将场景转移到移动端之后呢？

由于移动终端的特殊性，移动浏览器对开发者并不是十分友好。特别是遇到只能在真机上重现的Bug时，为了追查这个问题，只能通过逐行alert、修改->发布->刷新验证的方式，非常麻烦。

比较方便的开发流程是使用Fiddler工具，在局域网内，连接Fiddler、开发机、移动终端进行开发。Fiddler充当终端的HTTP代理，可以进行远程抓包，设置Host，对规则内的请求进行AutoResponse。

但是，使用以上方法之后，还是无法解决真机调试的问题。

##解决方案
> 这里讨论的调试，暂时只局限于**打印日志**这种简单粗暴的方式。类似与chrome开发者工具提供的断点调试、元素查看等方式太复杂，暂不讨论。

打印日志这种调试方式，是最原始，同时也是最简单有效的调试方式。**我们可以将log信息放在一条HTTP请求的包体中，然后通过Fiddler来解析并查看请求中的这条信息。**

##具体实现
* 客户端引入**fc.js**, 调用`Fc.log('print something')`，发出一条请求http://fiddler.fc.com;
  请求的域名是fiddler.fc.com，这是一个不存在同时也不应该存在的域名；
* Fiddler插件**FiddlerConsole**捕捉到这条session，获取session中包体的log内容，解析成json的格式，存到临时文件中；
* 使用Fiddler的autoResponder功能，将上面步骤中产生的临时文件作为该请求的响应。这样并不会因为域名不存在而返回错误的HTTP状态码；
* 关闭Fiddler时，删除打印log产生的一些临时文件，避免文件越积越多；
* 制作插件安装包，方便使用者安装插件。

###fc.js
实现一个ajax方法，用于发起一个http请求，如果在没有使用Fiddler的情况下使用，该请求会失败，然后设置此功能不可用，下次调用Fc.log时，直接返回。这样可以减小无效请求对页面性能的影响。代码片段如下：
{% highlight javascript %}
fc.log = function(){
    if(!fc.enable){
      return;
    }
    if(arguments.length < 1){
      return;
    }
    var data = parseargs(arguments);
    ajax({
      url : FC_URL + '?flag=' + data.flag,
      data : {value : data.value},
      fail : function(){
        fc.enable = false;//可能不在Fiddler环境下运行，需要禁用此功能
      }
    });
};
{% endhighlight %}

###FiddlerConsole插件
Fiddler插件在.net平台使用C#语言开发，Fiddler官方提供了[开发文档](http://docs.telerik.com/fiddler/extend-fiddler/extendwithdotnet)。网络上Fiddler插件的开发资料相对而言比较少，需要反复查询官方文档。
在FiddlerConsole插件开发的思路比较简单，需要注意的有三点：
1. 将log请求中的内容转成临时文件存储在本地；
2. 将Fiddler的`session['x-replywithfile']`赋值1中文件的文件名；
3. 在返回的响应头中添加`Access-Control-Allow-Origin:*`；
   添加这个之后，所有站点与fiddler.fc.com都能完成跨域请求。Fiddler插件充当一个服务器，相加啥就加啥，而且绝对安全~
4. 关闭Fiddler的时候，删除插件产生的所有临时文件

代码片段如下：
```c#
public void AutoTamperRequestBefore(Session oSession)
{
    if (oSession.host == HOST)//会话为console请求
    {
        var r = new Regex(@"flag=\w+", RegexOptions.IgnoreCase);
        var flagM = r.Matches(oSession.PathAndQuery);
        var flag = "";
        if (flagM.Count > 0) 
        {
            flag = flagM[0].ToString().Replace("flag=", "");
        }
        var data = Uri.UnescapeDataString(System.Text.Encoding.UTF8.GetString(oSession.RequestBody)).Replace("value=", "");
        if (flag != "")
        {
            data = "{flag:'" + flag + "',data:" + data + "}";
        }
        var filename = DateTime.Now.ToString("yyyyMMddHHmmssffff") + ".json";
        var path = TEMP_PATH + filename;
        oSession["ui-backcolor"] = "black";
        oSession["ui-color"] = "white";
        System.IO.File.WriteAllText(path, data);
        oSession["x-replywithfile"] = AppDomain.CurrentDomain.BaseDirectory + path.Replace("/", @"\");
    }
}

public void AutoTamperResponseBefore(Session oSession)
{
    if (oSession.host == HOST)//会话为console请求
    {
        oSession.oResponse["Access-Control-Allow-Origin"] = "*";//允许跨域访问
    }
}

public void OnBeforeUnload() 
{
    //清空临时文件
    Array.ForEach(Directory.GetFiles(TEMP_PATH), File.Delete);
}
```
###制作安装包
插件的最终形态实际上是一个dll文件，安装包所做的工作只是将这个dll文件放到Fiddler安装目录的Scripts目录下，安装包使用inno setup制作，具体的使用方法不在此处展开。
安装包同时需要具备以下功能：
* 检测是否安装了Fiddler
* 检测Fiddler是否处于运行中
* 获取Fiddler安装目录，将插件安装到`Fiddler安装目录\Scripts`中

###参考
http://www.therealmofcode.com/2014/03/creating-fiddler-extension-for-custom-filtering/
http://docs.telerik.com/fiddler/extend-fiddler/extendwithdotnet 

###项目主页
https://github.com/aslinwang/fc
