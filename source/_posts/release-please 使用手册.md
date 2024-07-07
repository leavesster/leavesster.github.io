---
title: release-please 使用手册
date: 2024-07-07
tags: opensource
---
  
[googleapis/release-please](https://github.com/googleapis/release-please) 可以在根据 [Conventional Commit messages](https://www.conventionalcommits.org/) 格式的 git commit 生成 changelog 的同时，更新版本号，生成 `github pull request`，打 tag 以及 GitHub release 为一体的工具。它与 github 强绑定，属于 GitHub 工作流工具。
## 使用准备
配置 github token 以及相关权限
	github Action 中，可以使用 `secret.GITHUB_TOKEN`，需要配置 github token 的 `contents` 和`pull-request`  的写权限。
[github Action](https://github.com/google-github-actions/release-please-action)
	在实际使用中，由于 cli 操作过于繁琐（每次都需附带 token 和 repo url），建议使用 GitHub Action 进行自动化操作。
	> 注意：使用 GitHub 提供给 Action 的 github.token 时，Action 的所有操作，都无法触发其他的 Action。  
	因此如果想触发其他的 workflow 需要使用 PAT。 fine-grained PAT 需要提供 `Contents` 和 `pull requests` 权限。 相关权限参考[文档](https://github.com/google-github-actions/release-please-action?tab=readme-ov-file#github-credentials)。  
配置好 [config 文件](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md#configfile) 和 [manifest](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md)
	必须指定好正确的 release-please，不同的 release-type 会决定工具查找版本文件的逻辑。
	手动配置 manifest 而不使用 cli 的 bootstrap 命令。在 cli 里指定 release-type 会直接按照单 repo 生成。
## monorepo 配置
monorepo 每一个 package 可以拥有独立的 changelog 和 Package version。更新日志也只会涉及当前 package 的 commit。
虽然文档中提到 monorepo 的支持可以通过[配置 manifest 来支持](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md)，但是具体操作语焉不详。最后参考微软博客 [Streamlining Development through Monorepo with Independent Release Cycles](https://devblogs.microsoft.com/ise/streamlining-development-through-monorepo-with-independent-release-cycles) 以及对应 [repo](https://github.com/MitaWinata/monorepo_independent_releases/blob/main/.github/workflows/release.yaml) ，以及参考 [npm/cli](https://github.com/npm/cli) 进行配置。
所有的配置文件，可以在 [monorepo-changelog-example](https://github.com/leavesster/monorepo-changelog-example) 里面查看。
### 操作步骤
自己写 manifest ，根据 package path 填好已有的版本号。
  在 config.json 里面写好每一个 package 的内容。  
如下示例：
```javascript
  # manifest，默认文件名 .release-please-manifest.json
  {
    # 路径而不是包名
    "packages/a": "0.2.0",
    "packages/b": "0.2.0"
  }
  
  # config，默认文件名 release-please-config.json
  {
    "packages": {
      # 路径
      "packages/a": {
        "component": "b" # 重命名包名，打 tag 和 pr 内容时使用
      },
      "packages/b": {
        "component": "a"
      }
    },
    "release-type": "node", # 决定查找版本号的文件地址
    "plugins": ["node-workspace"],
    "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json"
  }
```
## 更新版本
### 版本号更新逻辑
semantic version 的基础 `major.minor.patch`。
默认根据 [Conventional Commit messages](https://www.conventionalcommits.org/) 格式中的 fix 和 feat 两个 types 进行版本更新。
fix 更新 minor；feat 更新 patch；使用`fix!`, `feat!`,`refactor!`时，会更新 major。参考 [如何编写 commit Message](https://github.com/googleapis/release-please?tab=readme-ov-file#how-should-i-write-my-commits)。
> 版本更新策略（`versioning`字段，值参考[文档](https://github.com/googleapis/release-please/blob/main/docs/customizing.md#versioning-strategies)）可以指定固定更新 `patch`/`minor`/`major` 等行为，也可以根据文档自己实现。  
### 使用 Release-As 发布指定版本
除了使用 conventional commit 触发更新外，还可以在 commit body 使用 [release-as: x.y.z (大小写不敏感)](https://github.com/googleapis/release-please?tab=readme-ov-file#how-do-i-change-the-version-number) 信息发布指定的版本。
#### monorepo
对于 monorepo，`release-as` 指令会先跟随原有逻辑，根据 commit 扫描出改动的 package，然后使用 release-as 指定的版本进行更新。
	[测试 pr 2 更新所有 package 的版本](https://github.com/leavesster/monorepo-changelog-example/pull/2)
	[测试 pr 4 只更新涉及到 Package 的版本](https://github.com/leavesster/release-please-monorepo-example/pull/4)
如果是空 commit 在 monorepo 应该会触发所有包的更新（未测试）。
使用`config.json`各个 package 的 `release-as`指定每个包的版本。每个 commit body 单独写不同的版本比较麻烦，也可以直接在 manifest 的每个 Package 里面写上 release-as 配置，指定下次发布的版本号。不过随之而来是下次需要单独删除。
> 这个配置的优先级低于 cli 下直接使用 release-as 参数，但是不确定与 commit body 中 release-as 的优先级。  
## pr 标记
如果 pr 列表不存在标签`autorelease: pending`   or   `autorelease: triggered` 的 pr，就会创建一个新 pr。
如果存在`autorelease: pending` 的 pr，则会去更新这个 pr（github Action 中的表现）
新增的 pr 更新的版本号逻辑：有 break change 升级 major；有 feat 升级 minor；有 fix 升级 patch。
## tag 逻辑
目前打 release-please 打 tag 的逻辑：
	单 repo，直接是 v{major.minor.patch}
	monorepo：{componet}-v{major.minor.patch} component 名称。
tag 格式的配置，例如分隔符（默认`-`），`v`前缀配置。
前者可以在 config 中进行配置，字段为`tag-separator`，后者配置项为`include-v-in-tag`。
> 注意 tag separator 支持的格式有限，最好使用在release-please 仓库的 `test/util/tag-name.ts` 里支持的格式。  
