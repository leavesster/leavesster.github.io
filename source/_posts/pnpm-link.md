---
title: pnpm link 使用手册
date: 2024-01-30
tags: pnpm
---

主要用于调试跨 repo 的项目依赖，功能与 npm link 类似，但是参数和行为有所不同。本文是一些官方文档的翻译+一些注意事项和关联功能。
`-C/--dir`参数，可以修改`link`和`unlink`的影响目录（可以认为就是改变工作目录）
## [link](https://pnpm.io/cli/link)
link 的 api 不多，~~但是官方文档写得过于简单糟糕，以至于不明所以~~（有更新比较具体的用例）。同时文档里，没有提及如何查询 link 到 global node_modules 的包（实际上有，下文会提及）
```shell
  # 将 <dir> 里面的 package link 到当前工作目录（可以通过 --dir 参数更改）
  pnpm link <dir>
  # 将当前工作目录（或者`--dir`指定的目录）中的 package，link 到 global node_modules 中。
  pnpm link --global
  # 将 global node_modules 中的 `pkg` link 到当前目录（或者`--dir`指定的目录中）的 node_modules
  pnpm link --global <pkg>
```
### 状态查看
对于 link 到当前目录的操作，可以使用 `pnpm ls` 查看。
对于 link 到 global `node_modules` 里面的 package ，可以通过`pnpm ls -g` 查看。
其中的地址都会带上 link 前缀。
### 注意事项
workspace 下，需要在对应 package 里进行 link，在根目录 link 只会修改根目录的 node_modules。
link 是一次性行为，建立的是文件夹级别的软链接（因此 node_modules 是会被带过去）。
install 时，如果 link 的 package 在 package.json 里，则 link 会被移除，替换线上内容。
	如果 link 的包不在 package.json 声明，则 link 不会被移除。
## [unlink](https://pnpm.io/cli/unlink)
```shell
  # 移除当前目录下所有由 pnpm link 创建出来的 link，然后重新 install（yarn 不会做后面这一步）。
  pnpm unlink (in package dir)
  # 文档没有提及，cli 命令有提及，但是没有说明行为。猜测是 在当前目录 unlink 单个 pkg
  pnpm unlink <pkg>
```
在 workspace 里面需要使用`-r`参数递归子目录，同时可能需要使用`-w`表示在 root workspace。
### 移除 global node_modules
使用`pnpm link --global <pkg>` link 在 global `node_modules` 中的 package，需要使用`pnpm uninstall --global <package>`。unlink 只能移除当前目录的链接。
## link 与 file: 协议差别
可以查看 [官方文档](https://pnpm.io/cli/link#whats-the-difference-between-pnpm-link-and-using-the-file-protocol) 详细区别。主要差别在于是否处理这个 package 的依赖以及链接方式。
link 是对 package 做软链接，同时不会安装 package 的依赖（我理解是因为 link 可以不写在 package.json 的 dependencies 里面，只是单纯将 package 通过软链接的形式放到 node_modules 中）；file 是硬链接（提取对应内容，不会同步 link 其中的 node_modules），同时会安装依赖（因为 file 需要写在 package.json 里）。
> npm 对于 file 是复制，无法将 flie 对应 package 里的实时改动，体现在当前项目。  
