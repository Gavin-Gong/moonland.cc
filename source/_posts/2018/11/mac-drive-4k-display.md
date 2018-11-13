---
title: Macbook Pro 外接 4K 显示器
widgets: []
date: 2018-11-13
tags:
  - Mac
  - 显示器
categories:
  - Mac
---

双十一忍不住剁手买了台 4K 显示器，于是就有了这篇文章。

## 步骤一

禁用 SIP，先重启 Mac，摁住 Command + R 直到出现 Apple logo，松开进入 recovery mode，进入菜单栏找到 terminal，输入以下等待重启。

```bash
csrutil enable; reboot
```

## 步骤二

到这个[repo][1]中根据自己的 mac OS 版本下载对应的 patch 文件到本地，macOS \>= 10.12，下载`CoreDisplay Patcher`，否则下载`IOKit Patcher`用 `chmod +x`修改成可执行权限后执行。然后重启电脑。

<!-- more -->

## 步骤三

这个时候外接显示效果应用不会糊了。可以调整成想要的分辨率。也可以到 About This Mac → System Report → Graphics/Displays 查看外接显示器的分辨率是不是正常。没有缩放的情况下会显示显示器实际分辨率。

## 参考

- https://9to5mac.com/2016/06/04/how-to-enable-4k-60hz-resolution-2016-macbook/

- https://github.com/Floris497/mac-pixel-clock-patch-V2

[1]: https://github.com/Floris497/mac-pixel-clock-patch-V2 "repo"
