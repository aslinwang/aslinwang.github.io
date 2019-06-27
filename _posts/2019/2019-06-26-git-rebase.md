---
layout: post
title: 多人协作中git rebase的使用
categories:
- 前端
tags:
- git
---

## 使用背景
在多人协作的场景中，经常会出现多人在一个分支上开发，一般各开发者会基于这个分支拉一个本地分支进行开发，然后合并到共同分支提交。如果采用简单的git pull、git merge、git push，会导致提交日志出现混乱的菱形。如图：

![](http://blog-1253233020.cosgz.myqcloud.com/20190626161033.png)

## 实际操作
前提： 多人在dev分支共同开发，个人负责模块A开发，本地建立feat-A分支

1：如需更新dev分支的代码到feat-A分支(实际开发过程中需要及时更新，避免大面积冲突)：
不需要在commit之前执行git pull origin dev --rebase，可随时执行，执行时实际上是把dev分支合并到feat-A分支上，并且将feat-A的提交追加在dev分支最新提交的后面

{% highlight bash %}
~/demo on feat-A
$ git pull origin dev --rebase
{% endhighlight %}

执行以上命令的过程中如果遇到冲突，在解决冲突后可执行:
{% highlight bash %}
~/demo on feat-A
$ git rebase --continue
{% endhighlight %}

执行git rebase --continue后，可能还有冲突，继续解决冲突，执行git rebase --continue直到无冲突为止

![](http://blog-1253233020.cosgz.myqcloud.com/20190627124114.png)


2：合并feat-A分支到dev分支上

{% highlight bash %}
~/demo on dev
$ git pull origin dev --rebase
$ git merge feat-A
$ git push origin dev
{% endhighlight %}