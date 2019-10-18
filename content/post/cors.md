---
title: "cors跨域资源共享漏洞"
date: 2019-10-18T14:28:05+08:00
draft: false
tags: ['cors']
categories: ['web漏洞']
---
复习哈漏洞
<!--more-->
# 同源策略
## 0x01 什么是源
源（origin）就是协议、域名和端口号

## 0x02 什么是同源策略
同源策略是浏览器的一个安全功能，不同源的客户端脚本在没有明确授权的情况下，不能读写对方资源。所以a.com下的js脚本采用ajax读取b.com里面的文件数据是会报错的

# 怎样进行跨域
jsonp与cors两种标准
还有html标签也能跨域，有以下几种
img, iframe,ink, script

# cors流程
在配置了cors的前题下,当你登录网站A，并跨域访问网站B的时候，浏览器判断你的操作是跨域，这时候会在数据包里面加个orgin字段，内容为：orgin:b.com ,这样你就能跨域了。

# 漏洞产生原因
前面说了，设置了orgin就能进行跨域访问。而如果配置cors不当的话，如Access-Control-Allow-Orgin: 的话，就有可能存在问题。
Access-Control-Allow-Origin这个字段的意思是允许跨域访问的网站，配置为 的话，默认就能跨域访问所有网站，也就是说orgin这个字段是可控的。当然仅仅有一个orgin字段还不行，这里要说下
Access-Control-Allow-Origin-Credentials这个字段。这个字段的意思是是否带cookie跨域访问网站,如果为true的话，就是允许带着cookie跨域访问b网站。所以说，如果攻击者把orgin:设置成自己搭建的网站，功能为接收受害者的cookie。而Access-Control-Allow-Origin-Credentials又设置成true,这时候我们就可以说存在漏洞了

## 漏洞利用条件
看到前面漏洞产生原因，主要是Access-Control-Allow-Orgin:kissing_heart: 和Access-Control-Allow-Origin-Credentials:true这两个问题。有人可能要问了，这两个成立的话就会存在漏洞？答案是否定的。因为当你跨域的时候，浏览器看到你这两个的设置，会认为是不安全的，就是会拦截你的操作。而Access-Control-Allow-Orgin绑定了一个网站，但是对值没进行一个校验，也就是攻击者抓包把orgin改了，而网站没进行校验，就存在漏洞

所以
```
Access-Control-Allow-Orgin:XXXXX.com
Access-Control-Allow-Origin-Credentials: true
---可能存在漏洞
```
```
Access-Control-Allow-Orgin:*
Access-Control-Allow-Origin-Credentials: true
--- 不存在漏洞
```
# 攻击流程
攻击者利用cors漏洞把A.com的orgin改成接收用户信息的在线脚本B.com，然后生成一个链接引诱受害者去点击，如果受害者正好登录了A.com 并且点击了这个链接，则会把cookie发送到B.com

# 实战演示
[![](https://ww1.yunjiexi.club/2019/02/27/DNCk4.png)](https://ww1.yunjiexi.club/2019/02/27/DNCk4.png)
首先是一个这样 的界面，然后我们抓包，发现有Orgin字段
![](https://ww1.yunjiexi.club/2019/02/27/DNwUb.png)
我们把orgin改为我的博客地址试试
![](https://ww1.yunjiexi.club/2019/02/27/DN7SL.png)
发现Access-Control-Allow-Orgin已经为我的博客地址并且Access-Control-Allow-Origin-Credentials: true。说明存在这个漏洞

因为要接收受害者的信息，这里我们要搭建个poc来接收，这里我就直接用米斯特的poc平台了。
首先把刚才测试的url复制一下，然后放到里面
![](https://ww1.yunjiexi.club/2019/02/27/DNTnF.png)
这里生成了poc。我们可以把它发给受害者，如果受害者点击了这个poc。信息就会被我们接收



# 漏洞修复
Access-Control-Allow-Origin-Credentials:设置为false。如果有需求设置成true的话，就对orgin进行一个校验