7---
title: 记一次 Content-Security-Policy 请求头的坑 
widgets: []
date: 2019-12-27
tags:
  - HTML
categories:
  - Misc
  - 水文
---

公司一个项目用的 webpack-dev-server 进行代理服务，所有资源包括html，css，js等文件以及接口都是通过它进行访问，我想把本地开发效果给测试同事看，然后我把 webpack-dev-server 的 host 设置成 `0.0.0.0`, 
然后有意思的来了，我通过 `localhost:8080`,`127.0.0.1:8080`,`0.0.0.0:8080`都可以访问，然而神奇的是通过`lan_ip:8080`却只能访问到html，其他资源请求都转化成 HTTPS 请求了。
<!-- more -->

在项目的 index.html 文件中找到下面的meta信息设置

``` html
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">
```

作用就是让浏览器把一个网站所有的不安全 URL（通过 HTTP 访问）当做已经被安全的 URL 链接（通过 HTTPS 访问）替代。这个指令是为了哪些有量大不安全的传统 URL 需要被重写时候准备的。
那么问题来了。
实际上它会在请求头中添加 `upgrade-insecure-requests: 1`

但是实际使用来看貌似对本机地址访问并不起作用(Chrome 测试)，对其他ip地址(包含局域网ip)或者网址有效。

## 参考链接

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/upgrade-insecure-requests
