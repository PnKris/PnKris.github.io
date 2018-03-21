---
layout: post
title: Jekyll的安装与部署
modified:
categories: 
description:
tags: [Jekyll]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
date: 2018-03-21T12:58:22+08:00
---

之前的虚拟主机跟域名都已经到期了，本来就没写什么东西，就懒得续费了，打算把blog搬到GitHub上，也算是geek一下，毕竟都是搞技术的嘛 ^V^

对于GitHub上部署个人静态blog，大多数人的选择都是hexo或者jekyll，也有其他的好像不多。我就试着捣鼓了一下jekyll。

由于Jekyll没有后台，所有的文章都是以md的格式存放在_posts目录下面的，所以我们在提交到github之前都需要在本地看看效果怎么样。这种情况下我们就需要搭建一套环境，当然也可以直接修改以后就提交到GitHub上面直接看效果。


##### 环境配置：

Ruby: 直接在官网上下载就好了，我这里下载的是ruby 2.3.3
DevKit:也是直接在官网下载即可，这两个都在同一个网页上，https://rubyinstaller.org/downloads/

1、安装ruby，然后查看一下版本号看看有没有安装成功
	
	ruby -v

2、双击解压DevKit，进入devkit目录里面执行下面命令：
	
	ruby dk.rb init

	ruby dk.rb install

这样我们就已经安装好ruby，rubygems，devkit

<!--more-->

接下来我们需要安装一下Jekyll:
	
	gem install jekyll

查看版本号，验证一下有没有安装成功：

	jekyll -v

到这里我们的环境已经配置成功了。下一步创建一个blog



##### blog站点创建：
	
继续使用命令好：
	
	jekyll new blogName

这样就可以在目录里面看到你的blog了，接下来把blog提交到github上面。

如果喜欢我的blog样式可以直接在github上面fork一下https://github.com/aron-bordin/neo-hpstr-jekyll-theme 这个项目，按照README里面的步骤进行就好了
可能会遇到一个问题：报tzinfo相关的错误，解决办法如下：

安装tzinfo：
	
	gem install tzinfo

修改配置Gemfile文件：
需要在Gemfile文件里面新添加一行内容：

	gem 'tzinfo-data'



如果有自己的域名，那你可以在根目录下面可以创建一个CNAME文件，把你的域名保存一下
到dns服务商那里解析配置一下，等10分钟左右，就好了。
我的域名实在Godaddy上注册的，域名解析实在DNSPOD上进行解析的，怕麻烦就没有备案了，哈哈。
