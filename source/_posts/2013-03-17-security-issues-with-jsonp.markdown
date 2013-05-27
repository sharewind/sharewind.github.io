---
layout: post
title: "security issues with jsonp"
date: 2013-03-17 14:22
comments: true
categories: jsonp security
---

jsonp 是常用的跨域请求调用方式，然而不加限制的jsonp请求将给系统带来致命的危险。

#### 安全隐患 
1. 可能被他人嵌入到页面中，从而实现DDOS请求攻击。  
2. 如果有修改数据的接口被开放为JSONP，数据有可能被恶意脚本修改。  
例如：访问这个页面就执行登录用户的发微博，加好友等操作。

#### 措施  
1. 限制JSONP的接口为只读数据请求。  
2. 如果需求修改数据的跨域请求，可嵌入一个iframe 页面，将将跨域请转化为站内请求。  
3. 通过服务端代理转发跨域请求，例如通过 nginx 在服务端转发请求。  
4. 可以通过flash 或 html5 进行跨域请求，限制jsonp请求来源refer的domain。  
5. JSONP 不能返回有安全性相关或用户相关的重要数据。  
6. 无论是返回普通html页面还是json数据，都要对输出内容里进行escape,防止XSS 跨站脚本攻击。  

参考资料

- [Is JSONP safe to use?](http://stackoverflow.com/questions/613962/is-jsonp-safe-to-use) 
- [使用新浪API接口编写微博蠕虫](http://i.wanz.im/2012/05/19/use_sina_api_to_write_a_microblog_worm/)