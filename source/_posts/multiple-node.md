---
title: 2022 年多版本 node 管理工具安装方式
date: 2022-05-08 02:10:00
tags: nodejs
---


社区通用的 node 多版本管理工具，有两个 —— [nvm](https://github.com/nvm-sh/nvm) 与 [tj/n](https://github.com/tj/n)。
## nvm
nvm 比较流行，他不依赖 node 本身，需要单独安装。

缺点：
1. nvm 最大的问题是 nvm 的脚本会拖慢 shell 的启动速度。网上针对这个的推荐措施：写死一个常用的 node 地址，再针对 nvm 命令搞一堆懒加载。但是整个优化过程比较麻烦，同时分散在各个文章博客中，并没有官方推荐。所以懒加载过程并不是特别友好。

优点：
1. 可以使用`nvm use`根据当前目录的`.nvmrc`文件，指定 node 版本，还可以通过 [shell 脚本](https://github.com/nvm-sh/nvm#deeper-shell-integration) 达到在不同目录，自动使用不同的 node 版本的效果。
2. npm 跟随 node 版本走，全局包自带隔离。这一点是`tj/n`目前唯一无法直接满足的。

## tj/n
tj/n 默认`npm -g install n`的安装方式，需要一开始就有一个全局的`npm`。同时默认配置的`n`，下载 node 时，路径在`usr/local`，会导致下载时，需要使用`sudo`。

优点：
1. 简单，速度快
2. npm 全局共用，不用多次安装全局（`n exec <vers> <cmd> [args...]` 的全局包是与n全局环境的隔离的。哪怕他们实际是同一个node。）

缺点：
1. npm 全局包共用，有的全局包会要求 node 版本，可能会有问题。

### 我的安装方式
根据官方给出的其他安装方式，以及实际体验效果，我选择以下安装方式，绕过需要提前安装`npm`的要求，同时绕过`sudo`限制：
1. 安装裸的 n
``` shell
brew install n
```
2. 在对应 shell 配置文件中(zsh `~/.zshrc`)，配置 n 的环境变量
``` shell
# 设置 tj/n 的下载路径，默认在 /usr/local 中，需要使用 sudo 安装。通过修改下载地址，可以避免权限问题
export N_PREFIX="$HOME/.n"
# 如果当前环境没有 npm，第一次安装 node 时，不能配置，否则没有全局 npm 可以使用。
# export N_PRESERVE_NPM=1
# 将 tj/n 添加到 PATH。放在前面是为了覆盖旧的 node 和 npm，如果没有这个需求，可以随便放。
export PATH="$N_PREFIX/bin:$PATH"
```
3. 使用 n 安装一个 node 版本，由于没有设置`N_PRESERVE_NPM`此时会自带一个 npm 。
``` shell
# 安装 lts 版本 node
n lts
```


使用以上的安装方式，不需要前置`node`和`npm`，环境配置干净，非常适合新机环境操作。

如果在安装`n`前，已经安装了node，存在全局`npm`，这完全可以使用`npm install -g n`的方式进行安装，不过此时要根据自身情况，决定是否通过设置`N_PRESERVE_NPM`来保留这个旧的全局的`npm`。

### tj/n 的工作方式：
1. 下载预编译好的 node 版本，复制到`$N_PREFIX/bin`的目录中，通过修改`PATH`的方式，覆盖 node命令的查找逻辑。
2. 如果没有设置`N_PRESERVE_NPM`变量，这个`$N_PREFIX/bin`目录，还会带一份`npm`和`npx`的软链接。
	1. 虽然 npm 有隔离，但是实际上全局包都是安装在同一个目录(非`win`系统，在 `$(npm config get prefix)/lib/node_modules`下，如果要添加到`bin`的话，则是`$(npm config get prefix)/bin/`)中，在`n`进行`npm`切换时，并不会操作这个目录。所以全局包并没有被隔离。
3. 所有下载的都会存放在缓存目录中，所有已下载的版本，可以通过`n which`和`n run`以及`n exec`方式进行调用。

## 目录级别定义 node 版本
`n auto`可以自动根据`.n-node-version`,`.node-version`,`.nvmrc`来指定对应目录的 node 版本。如果没有以上文件，还会查找`package.json`中的`engine`字段，通过`npx semver`命令来确认对应兼容的 node 版本。
### 自动切换
可以使用[avn](https://github.com/wbyoung/avn)来做目录 node 自动切换，该命令支持 `n`和`nvm`(未尝试使用该功能)
nvm 官方提供了一个轻量级`avn`实现，可以自动目录中的读取`.nvmrc`文件，来做到自动切换。