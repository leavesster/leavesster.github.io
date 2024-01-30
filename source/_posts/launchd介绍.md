---
layout:
title: launchd介绍
date: 2021-09-11 21:51:01
tags: macOS
---

launchd 是可以统一管理守护进程，应用，脚本的开源服务管理框架。

守护进程是在后台运行，不需要用户输入的一类程序。

* launchd 会区分守护进程（`daemon`）与代理进程(`agent`)。前者可以以`root`用户，或者以`UserName`字段中定义的用户运行；而代理进程，只能使用登录用户运行。

## 任务定义

守护进程（`daemon`）与代理进程(`agent`)的行为配置，都保存一个叫 property list 的`XML`中。
系统会根据其存储的地方，来判断守护进程（`daemon`）与代理进程(`agent`)。

>Job Definitions

|Type|	Location | Run on behalf of |
| --- | --- | --- |
|User Agents |	~/Library/LaunchAgents |	Currently logged in user |
|Global Agents | 	/Library/LaunchAgents |	Currently logged in user |
|Global Daemons |	/Library/LaunchDaemons |	root or the user specified with the key UserName |
|System Agents	| /System/Library/LaunchAgents |	Currently logged in user |
|System Daemons | /System/Library/LaunchDaemons |	root or the user specified with the key UserName |


定义文件主要有三个字段：

1. `Label`: 强制要求，对于 launchd 而言，该值需要具有唯一性。理论上来说，由于代理进程和守护进程的启动用户不同，所以他们可以有相同的值，但是不推荐。
2. `Program`: 程序的调用位置。
3. `RunAtLoad`: 该字段有多个子字段，定义任务什么时候运行，以及其他相关行为。

>`Disable`字段可以设置是否加载该任务。

* XML 示例：

```XML
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
	<dict>
		<key>Label</key>
		<string>com.example.app</string>
		<key>Program</key>
		<string>/Users/Me/Scripts/cleanup.sh</string>
		<key>RunAtLoad</key>
		<true/>
	</dict>
</plist>
```

launchd 支持超过 36 种不同的配置，他们大多数在 [launchd](https://www.launchd.info/) 的 Configuration 部分有详细解释。

### 加载逻辑

>Automatic Loading of Job Definitions

1. 系统守护进程(`Daemons`)加载：系统会自动加载 `/System/Library/LaunchDaemons`和`/Library/LaunchDaemons`的文件。然后根据`Disable`字段以及`重载数据库`中的内容，来决定是否加载。
2. 用户代理(`Agents`)加载：当用户登录后，系统会有专门为该用户启动的新 launchd 进程，该进程会扫描`/System/Library/LaunchAgents`，`/Library/LaunchAgents`和`~/Library/LaunchAgents`目录，同样基于`Disable`字段和`重载数据库`字段，来决定是否加载。

### 加载并非启动

>Loading vs Starting

加载任务，并不意味着启动任务。任务何时启动，只取决于任务定义中的时机。只有当`RunAtLoad`为 true 或`KeepAlive`被定义时，launchd 才会在加载时无条件启动。

## 常用配置

### RunAtLoad

加载时，自动启动，Boolean 字段

### KeepAlive

该配置有好几种选项，可以为Boolean，也可以为字典。
* true: 永远保持运行状态，如果进程被关闭，会自动重启。
* SuccessfulExit: 该字段有 true 和 false。true 表示，只有当正常退出（exit code 为 0）时，会重新启动；false 表示，只有当非正常退出时，才会重新启动。
* Crashed: true 表示，crash 时，才重启；false 表示，crash 时，不重启。
还有一些其他奇怪的字段。

## 相关应用——启动项

>本小节主要内容来自：[launchd](https://www.launchd.info/) operation 部分。

>普通启动项
>系统设置 —— 用户与群组 —— 登录项
>一般应用，主要是那些老实的应用，或者自己主动配置的，会在这里启动。

部分应用，会利用 launchd 配置启动项。launchd 对于 Job 的加载，是基于其配置文件的 Disable 字段以及自身数据库。所以可以简单的使用 launchd 命令去取消其对任务的加载，从而达到取消该启动项。

> Disable 字段有时候会被应用自动修改回去，不如通过将禁用状态记录到`重载数据库`中来的简单有效。

## 重载数据库

launchd 在准备加载 job 时，会先检查`Disable`字段。`Disable`的 job，不会被加载。不过该字段的值，可以在`override database`中覆盖。

>有效防止部分程序每次自动修改该值的行为。

### 记录Job启用状态

```shell
# 命令行下使用的是：`launchctl`，对守护进程 daemon 需要使用`sudo`。
# 永久禁用任务
launchctl unload -w ~/Library/LaunchAgents/com.example.app.plist
# 永久启用任务
launchctl load -w ~/Library/LaunchAgents/com.example.app.plist
```

> 在 OSX 10.10 Yosemite 以及后续系统中，Job 状态可以切换，但是无法简单地将这个 job 内容从`重载数据库`中删除（删除文件同样可以禁用）。

> The database for daemons is `/var/db/launchd.db/com.apple.launchd/overrides.plist` (OSX 10.5 – OSX 10.9) or `/var/db/com.apple.xpc.launchd/disabled.plist` (OSX 10.10 and later).

### 删除Job配置

该数据虽然可以覆盖，但是一旦覆盖后，就无法正常使用 job 中的字段，也会造成一定问题。

#### OSX 10.5 Leopard - OSX 10.9 Mavericks

```shell
# agents
host:~ user$ sudo /usr/libexec/Plistbuddy /var/db/launchd.db/com.apple.launchd.peruser.`echo $UID`/overrides.plist -c Delete:local.job
# daemons
host:~ user$ sudo /usr/libexec/Plistbuddy /var/db/launchd.db/com.apple.launchd/overrides.plist -c Delete:local.job
```

#### OSX 10.10 Yosemite and later

>如果是移除代理程序，则可能需要获取当前用户的 

```shell
user id
id -u
```

>根据 Apple 教程进入恢复模式。[launch](https://www.launchd.info/)-operation。

在 工具选项下，选择 terminal。

>以下方法，假设系统盘为`Macintosh HD`，如果有不同，需要进行修改.

```shell
# local.job 需要替换为需要删除 job 的 Label

# 删除代理进程
# 需要将 502 需要更换为实际的 UID
/usr/libexec/Plistbuddy "/Volumes/Macintosh HD/var/db/com.apple.xpc.launchd/disabled.502.plist" -c Delete:local.job

# 删除守护进程
/usr/libexec/Plistbuddy "/Volumes/Macintosh HD/var/db/com.apple.xpc.launchd/disabled.plist" -c Delete:local.job
```

* 步骤三：重启

>目前如果不是洁癖，可以单纯地对文件进行禁用。无需删除数据库中的内容。

## 修改系统级别 job 配置项（关闭 sip）

#### OS X 10.11 El Capitan and newer
OS X 10.11 El Capitan introduced System Integrity Protection (SIP). This feature provides an additional layer of security by protecting certain sytem files from modification even by root. In order to make changes to those protected files you have to disable SIP:

1. Restart your Mac
2. Hold down ⌘R before macOS starts up. Hold these keys until the Apple logo appears. After your computer finishes starting up, you should see a Desktop with a menu bar and a window offering several utility functions.
3. Select Terminal from the Utilities menu (the menu, not said window)
4. Type csrutil disable and press the ⏎ key.
5. You should see a message that System Integrity Protection has been disabled
6. Reboot by entering reboot and press the ⏎ key.

SIP is now disabled.

>It is recommended to enable SIP again. Follow the instructions above. The command to enable SIP is csrutil enable

#### macOS 10.15 Catalina
In addition to SIP macOS 10.15 Catalina separates the Operating System (OS) from the user data. The OS is mounted read-only, so even with SIP disabled you cannot make changes to system files. In order to mount the OS partition in read/write mode disable SIP (see above), open Termina.app and enter:

```
sudo mount -uw /
```

## 参考资料

1. [launchd](https://www.launchd.info/)
1. [Technical Note TN2083](https://developer.apple.com/library/archive/technotes/tn2083/_index.html)