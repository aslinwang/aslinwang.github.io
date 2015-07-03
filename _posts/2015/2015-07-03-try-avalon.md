---
layout: post
title: avalon试用体验报告
categories:
- 前端
tags:
- MVVM
- avalon
---

##avalon是什么鬼
avalon是一个简单易用迷你的MVVM前端框架，为解决同一业务逻辑存在各种视图呈现而开发出来的。作者是前端大神司徒正美。
avalon将前端代码彻底分为了两部分，视图的处理通过绑定实现，业务逻辑则更多集中在叫VM(ViewModel)的对象中处理。操作VM数据，就能自然的同步到视图。

##avalon的优势
一个绝对的优势就是降低了耦合，让开发者从复杂的事件处理中挣脱出来。其他优势如下：

* 使用简单，上手快。作者声称自己吃透knockout，angular，rivets API设计，avalon并没有太多复杂的概念，指令数量控制得当，基本覆盖所有jquery操作
* 兼容IE6+以及其他主流浏览器
* 向前兼容好，不会出现angular那种跳崖式版本更新
* 注重性能
* 不依赖其他框架或类库，总体文件行数不超过5000行
* 支持管道符风格的过滤函数，方便格式化输出
* 操作数据即操作DOM，对VM的操作都会同步到View和Model上去，尽可能减少DOM操作的代码
* 出了问题可以抓得住作者（这一条是正美在CSDN上说的，也算一个优势=.=）

avalon导论到此为止，下面是avalon试用体验。

##一个avalon使用实例
上点代码来看看avalon具体要怎么玩
{% highlight html %}
<!DOCTYPE html>
<html>
    <head>
        <title>avalon demo</title>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <script src="avalon.js" ></script>
        <script>
            var model = avalon.define({
                $id: "test",
                name: "World"
            });
            avalon.scan();
        </script> 
    </head>
    <body>
        <div ms-controller="test">
            {% raw %}
            <p>Hello,{{name}}</p>
            {% endraw %}
        </div>
    </body>
</html>
{% endhighlight %}
是的，就只是在页面中打印了个"Hello, World"。
有几点需要注意的是，{% raw %}"{{}}"{% endraw %}双花括号界定符是实现属性绑定，里面可以写插值表达式实现过滤器的功能。
avalon.define可以定义一个ViewModel，在里面完成属性赋值，方法定义。"$id"就是指令"ms-controller"指定的controller值，通过"$id"可以将DOM和ViewModel结合起来。
除此之外，类名操作、样式操作、循环操作、显示隐藏控制、数据联动、模板引用、事件处理及路由系统等多个特性。这些特性，本文并不会一一描述，但其中一些会在下一节相关部分中涉及。详细用法可参阅[司徒正美博客](http://www.cnblogs.com/rubylouvre/tag/avalon/default.html?page=1)

##avalon实战
最近在做一个腾讯汽车购车通新版本的购车计算器项目（广告植入：[购车通，帮您轻松购车!](http://auto.qq.com/mobile.htm)），主要内容是根据各种计算规则和数据选项，组合之后计算最终购车时需要付的款项。如图所示：
![设计图](http://78rdmv.com1.z0.glb.clouddn.com/calc.png)

项目特点在于，数据之间有联动；一项数据变动款项值重新计算；按钮状态与数据相关联，radio选中则有数据，否则数据为0；全款计算和贷款计算两边的数据同步……
以上也是项目的难点。按照传统的做法，事件绑定+DOM操作将是非常恶心的开发体验，光是数据同步和联动就需要不断的重复监听数据变动以及操作DOM进行数据更新。
但是，使用avalon之后，是完全不同的编码体验。下面结合具体的case来说。

* 数据与视图同步
{% highlight html %}
<li>
    <div class="logo"><i></i></div>
    <div class="label">
      <span>购置税</span>
      <span class="value">{% raw %}{{purchTax}}{% endraw %}元</span>
      <i></i>
    </div>
</li>
{% endhighlight %}
使用{% raw %}{{purchTax}}{% endraw %}方式绑定属性，程序中对purchTax的更改都会自动同步到视图中，无需进行DOM操作

* 数据与视图同步(Hack)
{% highlight html %}
<span class="value">{% raw %}{{baseFee()}}{% endraw %}元</span>
{% endhighlight %}

{% highlight javascript %}
// 基本费用=购置税+上牌费+车船税+交强险
that.baseFee = function(vm){
  if(vm.price == 0){
    // hack：虽然这里计算了这堆数据，然而并没有什么用。感觉是avalon需要将baseFee方法与vm的属性动态建立联系
    var tmp = vm.purchTax + vm.licenseFee + vm.travelTax + vm.SALI;
    return 0;
  }
  return vm.purchTax + vm.licenseFee + vm.travelTax + vm.SALI;
}
{% endhighlight %}
在计算基本费用baseFee的时候，有一种情况需要处理，就是如果裸车价格为0，则基本费用也应该为0.但是，如果直接返回0的话，当裸车价格不为0之后，基本费用并不会同步计算。但是如果加上tmp = vm.purchTax + ...之后，就会同步计算。个人认为，是因为avalon在第一次计算的时候，会记住属性之间的依赖，如果只是单纯返回0，没有tmp那一句，avalon会认为baseFee不依赖其他属性，下次这些属性变化之后，就不会同步给baseFee

* 数据联动
{% highlight javascript %}
_avalon.$watch('price', function(){
    var vm = _avalon;
    vm.purchTax = Calc.purchTax(vm);// 车辆购置税

    if(!isRadioDisabled('.damageIns')){
      vm.damageIns = Calc.damageIns(vm);// 车辆损失险
    }
    if(!isRadioDisabled('.stolenIns')){
      vm.stolenIns = Calc.stolenIns(vm);// 全车盗抢险
    }
    if(!isRadioDisabled('.glassIns')){
      vm.glassIns = Calc.glassIns(vm);// 玻璃单独破碎险
    }
    if(!isRadioDisabled('.fireIns')){
      vm.fireIns = Calc.fireIns(vm);// 自燃损失险
    }
    if(!isRadioDisabled('.scratchIns')){
      vm.scratchIns = Calc.scratchIns(vm);// 车身划痕险
    }
});
{% endhighlight %}
通过$watch API对price属性进行监听，如果price有变动，则触发计算跟price相关的购置税及保险费用

* 类名操作、样式操作、事件绑定
{% highlight html %}
<body class="calc ms-controller" ms-class-loan="loan" ms-class-fullpay="fullpay" ms-controller="root" ms-on-swiperight="swiperight" ms-on-swipeleft="swipeleft" ms-on-touchmove="scroll" ms-css-height="pageH">
</body>
{% endhighlight %}
    
类名操作：ms-class-loan="loan"，当ViewModel的loan属性为true时，body上的class会增加"loan"类。这这里可以通过这种方式，控制页面显示贷款视图还是全款视图

样式操作：ms-css-height="pageH"，给ViewModel的pageH属性赋值之后，body上会增加样式"heigth:{pageH}px"样式

事件绑定：ms-on-swiperight="swiperight"，定义ViewModel的swiperight方法之后，向右滑动页面，可以触发事件，调用定义的swiperight方法

avalon内置的这些指令，可以让ViewModel中只用专注相关属性和方法的赋值和定义，不用再编写addClass、dom.style.height、addEventListener等代码

* 过滤器
{% highlight html %}
<span class="price">{% raw %}{{prepayFee() | currency('&lt;em&gt;￥&lt;/em&gt;', 0) | html}}{% endraw %}</span>
{% endhighlight %}
prepayFee()方法用于计算贷款首付，管道符代表应用过滤器处理结果。这里用到的过滤器是货币过滤器和html输出过滤器，同时过滤器还支持参数传入。最终经过处理的价格由"200000"变成"￥200,000"

* 模板引用
{% highlight html %}
<!--全款基本费用-->
<li class="base" ms-class-fold="basefold1" ms-on-click="onBase1Click"></li>
<ul class="flist item" ms-include="baselist"></ul>

<!--贷款基本费用-->
<li class="base" ms-class-fold="basefold2" ms-on-click="onBase2Click"></li>
<ul class="flist item" ms-include="baselist"></ul>

<script type="avalon" id="baselist">
  <li>
    <div class="logo"><i></i></div>
    <div class="label">
      <span>购置税</span>
      <span class="value">{% raw %}{{purchTax}}{% endraw %}元</span>
      <i></i>
    </div>
  </li>
  <li>
    <div class="logo"><i></i></div>
    <div class="label">
      <span>上牌费</span>
      <span class="value">{% raw %}{{licenseFee}}{% endraw %}元</span>
      <i></i>
    </div>
  </li>
  <li>
    <div class="logo"><i></i></div>
    <div class="label right" ms-on-click="onTravelTax">
      <span>车船税</span>
      <span class="value">{% raw %}{{travelTax}}{% endraw %}元</span>
      <i></i>
    </div>
  </li>
  <li>
    <div class="logo"><i></i></div>
    <div class="label right" ms-on-click="onSALI">
      <span>交强险</span>
      <span class="value">{% raw %}{{SALI}}{% endraw %}元</span>
      <i></i>
    </div>
  </li>
</script>
{% endhighlight %}
因为页面分为全款和贷款两个视图，而基本费用这一块的内容是一样的。所以将其提取出来，作为模板，在两个视图中分别引用。这样的好处有两个：一是相同结构公用一份模板代码，便于维护和修改；二是将模板的controller写在body层级上，在一个视图上的操作之后的数据变更立马同步到另一个视图之上。这一点正好可以满足产品对于全款计算和贷款计算两边的数据同步的要求。

使用avalon之后，编码工作主要集中在对ViewModel的属性和方法的赋值和定义工作上，数据绑定、事件监听等工作在html中完成。从以往繁杂的事件监听和处理以及DOM操作中脱离出来。avalon还有模块管理、路由系统等强大的特性，可以用来构造SPA项目，本次项目中没有用到。

##avalon的使用场景
在项目启动之初，也考虑过传统的zepto方案，但是看着复杂的界面交互和数据计算就放弃了。正好当前正接触avalon，所以就打算试用下。avalon比较适合这种强交互以及数据与视图相互关联的项目，比如内容管理系统之类的。对于偏重展示类型的页面和项目，实际上没有太大必要使用avalon。

虚拟DOM框架react目前正大火，司徒正美也在着手要在avalon中引入虚拟DOM技术，代号为[avalon2](https://github.com/RubyLouvre/avalon2)，相信融合了react之后的MVVM框架，会给以后的开发带来另一种编程体验。

##参考
[avalon官网](http://avalonjs.github.io/)

[avalon学习教程系列](http://www.cnblogs.com/rubylouvre/tag/avalon/default.html?page=1)