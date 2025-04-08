---
title: 我的首个GitHub Action诞生记：让EdgeOne缓存刷新更优雅
description: ''
date: 2025-04-08T15:31:14+08:00
lastmod: 2025-04-08T15:31:14+08:00
draft: false
---
> 每次在Hexo博客更新文章后，总要手动刷新腾讯云EdgeOne的CDN缓存才能看到最新内容——这个重复操作让我萌生了开发自动化工具的想法。起初只是在个人博客的GitHub Actions流程里硬编码了几行脚本，直到某天发现我的博客发布Action已经膨胀到将近300行代码，才意识到是时候把一些功能抽象成独立的Action了。首当其冲，先将刷新腾讯云EdgeOne的功能做成一个独立Action



我先了解了关于发布 Github Action 项目的一些要求：


> 以下内容摘录自[Github文档](https://docs.github.com/zh/actions/sharing-automations/creating-actions/publishing-actions-in-github-marketplace)

```
操作立即发布到 GitHub Marketplace，只要符合以下要求，就不会受到 GitHub 审查：

操作必须位于公共存储库中。
每个仓库必须在根目录中包含一个单一的操作元数据文件（action.yml 或 action.yaml）。
仓库可以在子文件夹中包含其他操作元数据文件，但这些文件将不会自动列在市场上。
每个存储库都不得包含任何工作流文件。
操作的元数据文件中的 name 必须是唯一的。
name 与 GitHub Marketplace 上发布的现有操作名称不匹配。
name 与 GitHub 上的用户或组织不匹配，除非用户或组织所有者正在发布操作。 例如，只有 GitHub 组织可以发布名为 github 的操作。
name 与现有的 GitHub Marketplace 类别不匹配。
GitHub 将保留 GitHub 功能的名称。
```

总结一下：除了名称不在重复等问题，需要注意的点就是不能有工作流文件，需要是公共仓库，最重要的需要一个元数据文件（action.yml 或 action.yaml），除此之外开发与普通脚本编写没有什么区别。


元数据文件action.yml与action.yaml的标准规范为在GitHub上的文档在：[GitHub Actions 的元数据语法 - GitHub 文档](https://docs.github.com/zh/actions/sharing-automations/creating-actions/metadata-syntax-for-github-actions)
