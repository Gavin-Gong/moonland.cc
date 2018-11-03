---
title: nginx 笔记
widgets: []
tags:
  - Nginx
  - Server
categories:
  - BE
---

## macos 安装

> brew install nginx

然后它的默认配置存放于 `/usr/local/etc/nginx/nginx.conf`
默认网站托管文件存放于 `/usr/local/Cellar/nginx/1.15.0/html`

## 基本操作

### 启动

> sudo nginx

<!--more-->

### 关闭

> sudo nginx -s stop

###  重载

> sudo nginx -s relaod

## 查看正在运行的 nginx 服务

> ps -ax | grep nginx

## 测试配置文件是否有效

> sudo nginx -t

## 场景

### 反向代理

> nginx: [emerg] unknown directive "proxy_pass:" in /usr/local/etc/nginx/nginx.conf:9

```nginx
events {
    worker_connections  1024;
}
http {
    include       mime.types; # 处理各种文件类型
    server {
        listen       80;

        location ~ /api {
            root /Users/me/projects/official-site/dist;
            proxy_pass  http://127.0.0.1:3000; // 一般使用这个就够了
            proxy_set_header Host $host:80;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            index index.html;
        }
    }
}
```

### SPA history 模式

```nginx
events {
    worker_connections  1024;
}
http {
    include       mime.types; # 处理各种文件类型
    server {
        listen       80;
        location / {
            root /Users/me/projects/official-site/dist;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### gzip

在 response Header 中看到 `Content-Encoding: gzip` 就说明开启成功了

```nginx
events {
    worker_connections  1024;
}
http {
    include       mime.types; # 处理各种文件类型
    gzip on;
    gzip_min_length  1000;
    gzip_types       text/plain application/x-javascript text/css application/xml application/javascript application/json;
    server {
        listen       80;

        location / {
            root /Users/me/projects/official-site/dist;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### 日志

```nginx
events {
    worker_connections  1024;
}
http {
    include       mime.types; # 处理各种文件类型
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /Users/me/projects/official-site/dist/access.log  main; // 访问日志
    error_log  /Users/me/projects/official-site/dist/error.log; // 错误日志
    server {
        listen       80;

        location / {
            root /Users/me/projects/official-site/dist;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### UA 重定向

```nginx
events {
    worker_connections  1024;
}
http {
    server {
        listen       80;

        if ( $http_user_agent ~* "(Android|iPhone|Windows Phone|UC|Kindle|iPad)" ){
            rewrite  ^/(.*)$  https://baidu.com/$uri redirect;
        }
    }
}
```

### 二级域名域名

首先你得到域名购买商增加一条 A record， 然后把主机记录设置子域名前缀 然后

```nginx
events {
    worker_connections  1024;
}
http {
    server {
        server_name api.example.com
        listen  80;

        if ( $http_user_agent ~* "(Android|iPhone|Windows Phone|UC|Kindle|iPad)" ){
            rewrite  ^/(.*)$  https://baidu.com/$uri redirect;
        }
    }
}
```

## 参考

[初识 Nginx](https://lufficc.com/blog/nginx-for-beginners)

[Nginx 配置 - Gzip 压缩](https://www.jianshu.com/p/e0ff1e275e7f)

[Nginx 查看日志](https://www.cnblogs.com/x123811/p/6026666.html)

[nginx 配置二级域名](http://originalee.oschina.io/2017/05/05/nginx%E9%85%8D%E7%BD%AE%E4%BA%8C%E7%BA%A7%E5%9F%9F%E5%90%8D/)

[Installing Nginx in Mac OS X Maverick With Homebrew](https://medium.com/@ThomasTan/installing-nginx-in-mac-os-x-maverick-with-homebrew-d8867b7e8a5a)
