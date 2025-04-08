---
title: 我的首个GitHub Action诞生记：让EdgeOne缓存刷新更优雅
description: ''
date: 2025-04-08T15:31:14+08:00
lastmod: 2025-04-08T15:31:14+08:00
draft: false
---
> 每次在Hexo博客更新文章后，总要手动刷新腾讯云EdgeOne的CDN缓存才能看到最新内容——这个重复操作让我萌生了开发自动化工具的想法。起初只是在个人博客的GitHub Actions流程里硬编码了几行脚本，直到某天发现我的博客发布Action已经膨胀到将近300行代码，才意识到是时候把一些功能抽象成独立的Action了。首当其冲，先将刷新腾讯云EdgeOne的功能做成一个独立Action


我先了解了关于发布Github Action项目的一些要求：

```

```
