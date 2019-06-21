---
layout: post
title: vs code远程开发配置
categories:
- 前端
tags:
- vsc
---

## vsc远程开发
2019年5月3日，在PyCon 2019大会上，微软发布了VS Code Remote，这意味着在本地使用vs code几乎所有的功能进行远程开发成为可能。我们可以像本地使用vs code的编辑、查找、代码提示、各种插件一样，无差别的开发远程服务器上的代码，而且无需在本地存放源代码。目前，在体验远程开发特性是，在连接远端、打开远端目录的时候，有延时感，但是在5G正在来临的时代，相信这种开发方式的开发体验将与本地开发无差别。

本次微软发布了三款核心的全新插件，Remote-SSH、Remote-Containers、Remote-WSL，可以帮助开发者在物理或虚拟机、容器、和Windows Subsystem for Linux(WSL)中实现无缝的远程开发

以下配置步骤以Remote-SSH为例，依赖：
* MacOS
* ssh

## 配置步骤
### vs code安装Remote Development扩展
![](http://blog-1253233020.cosgz.myqcloud.com/20190621144542.png)

### 创建ssh key
创建ssh key，建议不设置密码
{% highlight bash %}
ssh-keygen -t ecdsa -b 521 -C "aslinwang"
{% endhighlight %}
将生成的公钥(id_ecdsa.pub)的内容复制到远端服务器的~/.ssh/authorized_keys中，如果文件中已有内容，直接追加在文件末尾即可

### 配置本地ssh
为了方便vs code连接服务器，需要对本地ssh做相关配置

1、如果服务器允许本地直接ssh登录
{% highlight bash %}
Host aslin
  HostName xxx.xxx.xxx.xxx # 远程服务器IP，~/.ssh/authorized_keys需要配置ssh公钥
  User root
  ForwardAgent yes
  IdentityFile /Users/aslinwang/.ssh/id_ecdsa # 本地ssh key
  ProxyCommand corkscrew 127.0.0.1 12679 %h %p # 本地网络代理
{% endhighlight %}

2、如果服务器需要通过跳板机登录（跳板机不用走本地网络代理）

本地需要安装ssh-pass
{% highlight bash %}
Host aslin
  HostName xxx.xxx.xxx.xxx # 远程服务器IP，~/.ssh/authorized_keys需要配置ssh公钥
  User root
  ForwardAgent yes
  IdentityFile /Users/aslinwang/.ssh/id_ecdsa
  ProxyCommand sshpass -p [跳板机密码] ssh -p [跳板机端口] root@[跳板机IP] -W %h:%p 2> /dev/null
{% endhighlight %}

3、如果服务器需要通过跳板机登录（跳板机需要走本地网络代理）

首先配置跳板机ssh
{% highlight bash %}
Host jumper
  HostName yyy.yyy.yyy.yyy # 跳板机IP，跳板机~/.ssh/authorized_keys需要配置ssh公钥
  User root
  ForwardAgent yes
  IdentityFile /Users/aslinwang/.ssh/id_ecdsa
  ProxyCommand corkscrew 127.0.0.1 12679 %h %p
{% endhighlight %}

然后通过跳板机访问服务器
{% highlight bash %}
Host aslin
  HostName xxx.xxx.xxx.xxx # 远程服务器IP，~/.ssh/authorized_keys需要配置ssh公钥
  User root
  ForwardAgent yes
  IdentityFile /Users/aslinwang/.ssh/id_ecdsa
  ProxyCommand ssh jumper -W %h:%p 2> /dev/null
{% endhighlight %}

### vs code连接服务器

打开Command Palette

![](http://blog-1253233020.cosgz.myqcloud.com/20190621151746.png)

选择要连接的服务器，这里出现的服务器都是在~/.ssh/config中配置好的

![](http://blog-1253233020.cosgz.myqcloud.com/20190621152010.png)

进入vs code远程编辑窗口

![50%](http://blog-1253233020.cosgz.myqcloud.com/20190621152356.png)

如上图所示，甚至可以编辑服务器的全盘文件，下面的终端窗口，可以直接使用命令行操作服务器