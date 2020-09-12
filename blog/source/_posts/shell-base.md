---
title: 编写 shell 的几个基础技巧
date: 2020-09-12
tags: unix, shell
---

## 语法检查

[shellcheck](https://www.shellcheck.net)
[shellexplain](https://explainshell.com/)

## [shebang](https://zh.wikipedia.org/wiki/Shebang)
shell 调用对应文件的解释器。
书写更通用的解释器查找: `#!/usr/bin/env 脚本解释器名称`

## 善用 set 命令

书写更人类友好安全的脚本
`set -euxo pipefail`

`-u`:有未使用的变量，就中断脚本
`-x`:在运行结果之前，先输出要执行的命令
`-e`:有报错就中断脚本
`-o pipefail`:补充`-e`例外情况：管道命令`|`

>`bash` 支持较为完整，`sh` 有部分不支持。


## 单双引号与转义

参考资料：https://harttle.land/2020/06/26/bash-quote-escape.html

* 反斜杠

转义符，使部分用来表达功能的特殊字符，显示原始内容而不是使用特殊表达。
反斜杠 + 换行符，表示继续输出内容，在比较长的命令中比较常见

* 单引号

禁止所有转义，所见即所得。

* 双引号

大部分保留字面内容，除以下`$`，`\``，`\`三种

1. $ 用来做 Bash 参数展开，比如 echo "my name is $name."。
1. ` 表示 命令替换，基本等价于 $()。
1. \ 是我们讨论的重点，它用来转义。

## 命令替换 `` 与 $()

执行其中的命令，将结果替换出来，组成新命令。

## 环境变量

```shell
a=ww
export b=ddd
unset b
```
没有`export`的是本地变量，有`export`的才算是环境变量，`unset`则是撤销。

## fork source exec

fork：子shell（输入 scripts 的地址，即为 sub-shell 中执行）
source：当前 shell 执行；
exec：在同一个进程执行，但是原有进程被终结（未验证）

## {} 与 ()

{}: 子进程 sub-shell
(): 同一进程 non-named command group

## $符号

变量 | 含义
------- | -------
$0 | 文件相对路径
$n | 传递shell的参数。n 是一个数字，表示第几个参数（<10）。例如，第一个参数是 \$1，第二个参数是 \$2 。0就是文件相对路径。
$# | 传递给脚本或函数的参数个数。
$* | 传递给脚本或函数的所有参数。
$@ | 传递给脚本或函数的所有参数。被双引号(" ")包含时，与 \$* 稍有不同，详情见参考资料。
$? | 上个命令的退出状态，或函数的返回值。
$$ | 当前Shell进程ID。对于 Shell 脚本，就是这些脚本所在的进程ID。

参考资料: [Shell特殊变量](http://c.biancheng.net/cpp/view/2739.html)

## 获取当前脚本目录

* [绝对路径](https://stackoverflow.com/questions/4774054/reliable-way-for-a-bash-script-to-get-the-full-path-to-itself): [answer](https://stackoverflow.com/a/4774063/4770006)

```shell
# 如果存在软链接，会跳转到脚本的真实路径
$(cd $(dirname "$0"); pwd -P)
# 脚本所在的目录，即使是软链接也不会跳转
$(cd $(dirname "$0"); pwd)
```

>Note also that esoteric situations, such as executing a script that isn't coming from a file in an accessible file system at all (which is perfectly possible), is not catered to there (or in any of the other answers I've seen).
>以上无法满足引用中的情况

* 考虑脚本地址为软链接：[question](https://stackoverflow.com/questions/59895/getting-the-source-directory-of-a-bash-script-from-within/246128#246128): [anser](https://stackoverflow.com/a/246128/4770006)(需要readlink，太过复杂……)

* [相对路径](https://stackoverflow.com/questions/242538/unix-shell-script-find-out-which-directory-the-script-file-resides):
`$(dirname "$0")`
