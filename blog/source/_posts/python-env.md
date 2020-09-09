---
title: python虚拟环境
date: 2019-03-28 23:05:54
tags: python
---

一般会有一个 requirements.txt，用来一次性安装需要的依赖。

```shell
# 将当前pip安装的依赖库全部输出在对应文件
pip freeze > requirements.txt
# 安装文件中的所有 python 库
# -r, --requirement <file>    Install from the given requirements file. This option can be used multiple times.
pip install -r requirements.txt
```

此处所有操作命令，都只写了 Linux/MacOS 的版本，未列出 Windows。（官方有仔细写出）

## [virtualenv](https://virtualenv.pypa.io/en/latest/)

**隔离 python 依赖包环境**。
能够让每一个项目，都有一个单独的 python 环境。支持各个 python 版本，但是与 python 版本耦合，需要单独安装。

```shell
# 安装 virtualenv
pip install virtualenv

# 创建虚拟环境
virtualenv ENV
# 激活虚拟环境
source /path/to/ENV/bin/activate
# 退出虚拟环境
deactivate
```

因为大多数时候，使用最新版本的 python，所以，很少使用。应该是最早的隔离工具，后续的工具，隔离python 依赖包环境时，基本都继承了这个三个方法。

## [venv（python 3.6+)](https://docs.python.org/3/library/venv.html)

**python3.6+ 官方，用来隔离 python 环境的工具**。

*python3.4开始，官方自带 `pyvenv`，但是3.4有问题。3.5正式支持，由名字过于容易混淆，在3.6中该名称(pyvenv)被弃用*

```shell
# 创建虚拟环境
python -m venv <VENV>
# 启动虚拟环境，linux/macOS
source ./env/bin/active
# 关闭虚拟环境
deactivate
```

>venv 出来的 pip，用的是 python发行时的 pip，不会使用后续更新的pip 版本。提示有点烦人。

## [Pyenv](https://github.com/pyenv/pyenv)：隔离python版本

是一个 shell 层的插件，不支持Windows。由于是 shell 插件，所以本体和插件安装完以后，需要修改 shell 配置文件来启用。具体参考官方。

**跟以上工具不同，这个工具，主要用来隔离不同的 python 版本** 。

[安装以及 shell配置](https://github.com/pyenv/pyenv#installation)
[Command Reference](https://github.com/pyenv/pyenv/blob/master/COMMANDS.md)

* 插件：[pyenv-virtual](https://github.com/pyenv/pyenv-virtualenv)：支持隔离各个 python 版本的依赖包环境。

pyenv 的一个插件，支持给 pyenv 的不同版本创建不同的虚拟包环境。否则，不同版本的 pip 安装都是装在对应版本的全局环境中。

[安装以及 shell 配置](https://github.com/pyenv/pyenv-virtualenv#installation)

```shell
# 激活环境
pyenv activate <name>
# 删除虚拟环境
pyenv uninstall my-virtual-env
# 退出虚拟环境
pyenv deactivate
```

## [conda](https://conda.io/docs/)

暂未了解

## [pipx](https://github.com/pipxproject/pipx)

```shell
# --user 把第三方库装进用户相关的路径，而不是全局
python3 -m pip install --user pipx
# 给 userpath 增加一个环境变量连接（可惜只是操作 bash，没检测到我有 zsh）
python3 -m userpath append ~/.local/bin
```

隔离环境下，安装执行`python`第三方库。

官方安装手册不是很完善，还是需要手动把环境变量搞好。

几个命令倒是好猜。但是这个更像是临时性质的，pipx 对每个库只缓存 2 天。