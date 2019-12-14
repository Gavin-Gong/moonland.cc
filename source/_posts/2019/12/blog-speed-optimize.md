---
title: Hexo Blog 访问优化备忘小记
widgets: []
date: 2019-12-14
tags:
  - 访问优化
categories:
  - Misc
---

之前使用 Netlify 进行自动部署应用，但是发现访问还是有些慢，索性花点心思把整个网站网络访问优化一遍。

## CDN 优化

将 `google font` 和 `font-awesome` 换成了国内的 `CDN`，jsdelivr 家的 `CDN` 国内访问速度还不错就懒得换了。

```yml
providers:
  cdn: jsdelivr
  fontcdn: https://fonts.loli.net/${type}?family=${fontname}
  iconcdn: https://lib.baomitu.com/font-awesome/5.4.1/css/all.min.css
```

<!-- more -->

## 缓存控制

Netlify 家构建后的资源资源文件是可以设置请求头的，所以给一些静态文件设置缓存头，依照我的更新频率，设置一年也问题不大 emmmm

```toml
[[headers]]
  for = "/js/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000"
[[headers]]
  for = "/css/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000"
[[headers]]
  for = "/images/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000"
```

## 代理

Netlify 家大陆访问有时候超级慢，但是又不想放弃它打包的便利（其实也不想把打包文件放 git 仓库，不然我就用 travis 了），然后用 cloudflare 中间代理加速访问，毕竟 cloudflare 甚至可以代理 SSR 流量，
步骤

- 将 namecheap（域名购买商）的 name server 设置为 cloudflare 提供的域名
- 一段时间生效后，由 cloudflare 接管域名的解析记录
- 将 Netlify CNAME + A 解析到 cloudflare
