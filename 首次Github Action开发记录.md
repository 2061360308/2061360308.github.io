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


## 项目介绍


本项目本质是一个Node项目，并使用rollup进行打包，另外还利用一些工具规范化提交和版本发布。



具体项目结构如下：

```tree
edgeone-purge-action
├── src/                   # 源代码
├── dist/                  # 编译输出
├── test/                  # 新增测试目录
│   ├── test.js            # 测试用例
│   └── test.yml           # 测试workflow
├── rollup.config.js       # 新增Rollup配置
├── commitlint.config.js   # 新增Commit规范配置
├── action.yml             # Action核心配置
└── package.json           # 项目依赖配置
```


给出Action的核心配置文件，在这个配置中规定了action的入口dist/main.js，执行环境node20，以及需要的一些输出参数。

```yaml
name: 'Tencent EdgeOne Purge Action'
description: 'Purge cache in Tencent EdgeOne'
author: '盧瞳/LuTong'

branding:
  icon: 'zap'
  color: 'blue'

inputs:
  secret_id:
    description: 'Tencent Cloud Secret ID'
    required: true
  secret_key:
    description: 'Tencent Cloud Secret Key'
    required: true
  zone_id:
    description: 'EdgeOne Zone ID'
    required: true
  type:
    description: 'Cache type'
    required: true
    default: 'purge_host'
    values: ['purge_host']
  hostnames:
    description: 'Comma-separated list of hostnames to purge'
    required: false

runs:
  using: 'node20'
  main: 'dist/main.js'
```

## @actions/core使用



在开发 GitHub Action 时， @actions/core 是必不可少的核心工具库，它提供了与 GitHub Actions 运行环境交互的关键功能。以下是需要特别注意的使用要点：

### 1. 日志输出规范

@actions/core 提供了分级的日志输出方法，应该根据信息重要性选择适当的方法：

```typescript
import * as core from '@actions/core'

// 调试信息 (仅在设置 ACTIONS_STEP_DEBUG=true 时显示)
core.debug('正在验证输入参数...')

// 普通信息 (始终显示)
core.info('开始执行缓存刷新操作')

// 警告信息 (黄色高亮显示)
core.warning('未指定hostnames参数，将刷新整个域名')

// 错误信息 (红色高亮显示，但不会终止流程)
core.error('解析JSON配置失败')
```

### 2. 错误处理机制

正确的错误处理对 Action 的可靠性至关重要：

```typescript
try {
  // 业务逻辑代码
} catch (error) {
  // 设置Action为失败状态并输出错误信息
  core.setFailed(`操作失败: ${error.message}`)
  
  // 同时记录详细错误堆栈
  core.error(error.stack)
}
```

### 3. 输入输出处理

@actions/core 提供了标准的输入输出处理方法：

```typescript
// 获取输入参数
const secretId = core.getInput('secret_id', { required: true })
const hostnames = core.getInput('hostnames', { required: false })

// 设置输出参数 (供后续步骤使用)
core.setOutput('purged_urls', purgedUrls.join(','))
```

### 4. 状态管理

对于长时间运行的任务，应该定期更新状态：

```typescript
// 开始执行任务
core.saveState('start_time', new Date().toISOString())

// ...任务执行中...

// 恢复状态
const startTime = core.getState('start_time')
core.info(`任务已运行 ${(new Date() - new Date(startTime)) / 1000} 秒`
```

### 5. 最佳实践建议

1. 重要操作前添加检查点 ：

```ts
if (!isValidZoneId(zoneId)) {
  core.setFailed('无效的Zone ID格式')
  return
}
```

2. 为关键操作添加分组 ：

```typescript
core.startGroup('刷新EdgeOne缓存')
// 刷新操作代码...
core.endGroup()
```

3. 添加必要的环境检查 ：

```typescript
if (process.env.GITHUB_ACTIONS !== 'true') {
  core.warning('当前不在GitHub Actions环境中运行')
}
```


## 本地测试



在开发 GitHub Action 时，测试环节往往比较困难。最直接的方法是编写 workflow 提交到 GitHub 仓库后手动触发运行，但这种方法存在两个主要问题：

1. 每次测试都需要提交代码到远程仓库
2. 测试运行需要等待 GitHub 的 runner 调度，耗时较长

### 本地测试工具 act 介绍

act 是一个开源工具，可以在本地运行 GitHub Actions，大大提高了开发效率。它的主要特点包括：

- 在本地模拟 GitHub Actions 运行环境
- 支持大部分 GitHub Actions 功能
- 可以快速迭代测试，无需等待远程 runner

### 安装 act

安装act之前要知道他通过Docker的方式运行工作流，所以你需要先配置安装号Docker环境。

之后act的安装也比较简单，从官网下载后放到指定目录配置环境变量即可。

[Releases · nektos/act](https://github.com/nektos/act/releases)


### 使用act

运行指定的workflow `act -W ./workflows/your_workflow.yml`

如果需要某些输入的话，可以通过.env文件来指定`act --secret-file .secrets -W ./workflows/test.yml`



## 提交规范化，以及版本发布工具

另一个我想记录的事情是本次耗费时间折腾的两个项目管理工具。



Github Actcon发布到市场需要发布版本，这么做可以更规范一些，还可以更为方便地自动生成CHANGELOG。


### Commitizen - 规范化提交工具


Commitizen 帮助开发者遵循 Conventional Commits 规范，为后续自动生成 CHANGELOG 打下基础。

#### 安装配置

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional @commitlint/cz-commitlint @commitlint/types
```

> Typescript项目需要`@commitlint/types`


三个包的作用如下：

- @commitlint/cli - Commitlint 的核心命令行工具
- @commitlint/config-conventional - 提供 Conventional Commits 规范配置
- @commitlint/cz-commitlint - 将 Commitlint 与 Commitizen 集成


在package.json中可以添加以下配置项：

```json
"config": {
    "commitizen": {
      "path": "@commitlint/cz-commitlint"
    }
  },
```

另外命令中可以配置下面命令

```json
"scripts": {
    "prepare": "husky",
    "commit": "pnpx git-cz"
}
```


#### 使用示例

```bash
git add .
npm run commit
```


系统会交互式引导你填写规范的提交信息，例如：

```plaintext
? Select the type of change that you're committing: 
  feat:     A new feature
  fix:      A bug fix
  docs:     Documentation only changes
  style:    Changes that do not affect the meaning of the code
  refactor: A code change that neither fixes a bug nor adds a feature
  perf:     A code change that improves performance
  test:     Adding missing tests or correcting existing tests
```

### 2. standard-version - 自动版本管理和 CHANGELOG 生成

standard-version 可以根据提交信息自动：

- 升级版本号 (遵循 SemVer)
- 生成 CHANGELOG.md
- 创建版本 tag

#### 安装配置

```bash
npm install --save-dev standard-version
```

在 package.json 中添加脚本：

```json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:patch": "standard-version --release-as patch",
    "release:major": "standard-version --release-as major"
  }
}
```


#### 使用流程

1. 开发完成后提交所有变更
2. 运行发布命令：

```bash
npm run release
```

3. 推送到远程仓库：

```bash
git push --follow-tags origin main
```

## 与 GitHub Marketplace 集成

规范的提交和版本管理使你的 Action 在 GitHub Marketplace 中更专业：

1. 每次发布新版本时，GitHub 会自动识别 tag 并更新 Marketplace
2. CHANGELOG.md 会显示在 Marketplace 页面，方便用户了解变更
3. 版本号遵循 SemVer，用户可以根据版本号判断兼容性

## 示例 CHANGELOG 输出

运行 standard-version 后生成的 CHANGELOG.md 示例：

```markdown
# Changelog

## [1.1.0] - 2023-11-15

### Features
- 新增对多域名同时刷新的支持 (#12)

### Bug Fixes
- 修复了 Secret 验证失败的问题 (#10)

## [1.0.0] - 2023-10-20

### Initial Release
- 实现基础缓存刷新功能
```

这套工具链能显著提升项目的专业度和维护效率，特别适合需要频繁发布到 GitHub Marketplace 的 Action 项目。
