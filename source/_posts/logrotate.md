---
title: logrotate配置手册
date: 2021-01-18
tags: Linux
---

# logrotate

一般情况下，Linux 下进程，都会将日志存储在`/var/log`文件夹。大多数发行版默认集成`logrotate`，用户通过配置，可以对日志进行切割，备份以及删除。

## rotate

`logrotate`的定时机制，依赖于`crontab`服务。logrotate 会在`/etc/cron.daily`下插入一份启动 `logrotate`的`crontab`配置，然后由`crontab`每天定时调用 logrotate。

>由此可以得知`logrotate`并不是 daemon，修改配置后，只需要等待下次启动，即可生效。

该`crontab`配置由于发行版和`logrotate`版本差异，可能略有不同，但是差异不大。以下为`Ubuntu`18.04.3 的配置：

```shell
cat /etc/cron.daily/logrotate
#!/bin/sh

#`Clean`non`existent`log`file`entries`from`status file
cd /var/lib/logrotate
test -e`status`||`touch`status
head -1`status`> status.clean
sed 's/"//g'`status`|`while`read`logfile`date
do
   `[`-e "$logfile"`]`&&`echo`"\"$logfile\" $date"
done >> status.clean
mv status.clean status

test -x /usr/sbin/logrotate ||`exit`0
/usr/sbin/logrotate /etc/logrotate.conf
```

* 由此带来的特点

logrotate 有基于文件大小进行切割的配置参数。但是`logrotate`只会在特定时间进行检查，所以并不意味着文件超出规定大小就会被分割转存。仍然需要在`logrotate`执行时，检查文件大小。

### crontab 配置文件

与`logrotate`类似，`crontab`也有默认配置以及自定义配置文件夹。`/etc/cron.daily`，其实是由`crontab`的默认配置中提供的每日执行位置。
具体可以查看`crontab`的全局配置文件`/etc/anacrontab`（部分旧版本为`/etc/crontab`）
通过该全局配置文件，得知具体`daily`等任务的具体执行时间。

## 分割原理

`logrotate`的日志分割有两种模式：`create`与`copytruncate`。

* create

`create`为默认方案，思路是重命名原日志文件，创建新日志文件，再通知进程重新打开日志文件，使其后续输出内容导向新文件。新文件会成为日志输出目标。

步骤如下：

1. 重命名当前的日志文件。由于重命名只是修改目录以及文件名；而进程使用的是`inode`进行操作，所以不影响原进程的日志输出。
2. 创建新日志文件，文件名为原日志文件名称。
3. 通过各种方式(scripts 相关配置参数)通知程序，重新打开日志文件。此后的日志，就会输出到新文件中。

好处：简单，但是有些程序不提供重开日志的接口，只能重启程序，这回降低可用性。

* copytruncate

思路：将文件中已有的日志内容拷贝出来，转存到的新日志文件，然后清空旧日志文件中的内容。新文件保存旧日志内容，日志输出文件不变。

1. 复制日志文件，根据配置规则命名文件名。原文件不动。
2. 将日志文件的之前的内容进行清空。

缺点：如果日志文件过大，复制会比较耗时，清空可能会导致部分日志丢失（也就是说，并没有标记复制内容的位置，只是复制完，对日志文件进行了清空操作）。
优点：对程序无感知。

## 配置文件

可执行文件位置：`/usr/sbin/logrotate`。
默认配置文件位置：`/etc/logrotate.conf`。
自定义配置文件在`/etc/logrotate.d/`文件夹（通过默认配置文件中的`include`来读取）。

* 默认配置大致为以下内容：

```shell
cat /cat /etc/logrotate.conf
#`see`"man logrotate"`for`details
#`rotate`log`files`weekly
weekly

#`use`the`syslog`group`by`default,`since`this`is`the`owning`group
#`of`/var/log/syslog.
su`root`syslog

#`keep`4`weeks`worth`of`backlogs
rotate 4

#`create`new (empty)`log`files`after`rotating`old`ones
create

#`uncomment`this`if`you`want`your`log`files compressed
#compress

#`packages`drop`log`rotation`information`into`this`directory
include /etc/logrotate.d

#`no`packages`own`wtmp,`or`btmp -- we'll`rotate`them here
/var/log/wtmp {
    missingok
    monthly
   `create`0664`root`utmp
   `rotate`1
}

/var/log/btmp {
    missingok
    monthly
   `create`0660`root`utmp
   `rotate`1
}

# system-specific`logs`may`be`configured here
```

logrotate 会根据执行时指定的配置文件路径，读取所有配置文件。每个配置文件，都可以有全局选项（本地的配置会覆盖全局的，新的配置，会覆盖旧的配置）和指定日志文件切割转储。

* 问题：覆盖问题的处理

1. 各个配置文件之间的全局配置，是否有覆盖。如果是覆盖，但是又无法控制文件加载顺序，会导致很多问题。
2. ~~使用`include`通配符时，配置文件的加载顺序~~（根据`include`定义，应该是根据字母表顺序加载）
3. 特定文件的配置，最终如何生成，是加载时，当时的全局配置+特定文件配置生成；还是在读取完所有配置后的全局配置+特定文件配置生成。（可以确认的是，如果特定文件的配置重复，会由新的覆盖旧的）

>1. 自定义配置的全局配置，应该会覆盖默认配置的全局配置。
>3. 如果随意在自定义配置中设置全局配置，应该会造成很多问题。大部分自定义配置，也的确只针对特定日志文件，进行配置，从而规避了前面的问题。

* 配置文件简单示例：

```conf
#`sample`logrotate`configuration`file
compress

/var/log/messages {
   `rotate`5
    weekly
    postrotate
        /usr/bin/killall -HUP syslogd
    endscript
}

"/var/log/httpd/access.log" /var/log/httpd/error.log {
   `rotate`5
   `mail`www@my.org
   `size`100k
    sharedscripts
    postrotate
        /usr/bin/killall -HUP httpd
    endscript
}

/var/log/news/* {
    monthly
   `rotate`2
   `olddir`/var/log/news/old
    missingok
    postrotate
       `kill`-HUP 'cat /var/run/inn.pid'
    endscript
    nocompress
}
```

第一行为全局配置，代表所有日志文件，在切割转储后，都会被压缩。

接下来的文件配置部分，定义了对`/var/log/messages`日志文件的处理。该日志文件，每周执行一次切割转储，最多保留五份切割后的文件。在日志文件完成切割转储之后，压缩旧日志文件之前，会执行`postrotate`和`endscript`中的命令内容。

接着的配置小节，定义了对`/var/log/httpd/access.log`和`/var/log/httpd/error.log`的切割转储处理。当日志达到 100k 时，执行切割。当切割文件超出保留数之后，解压后的文件，会发送`email`到 www@my.org 邮箱。`sharedscripts`配置表示，`postrotate`脚本，会在两个日志文件都切割转储并压缩后，才执行一次`postrorate`命令，而不是每个文件都执行一次。当文件路径以双引号包裹时，可以支持路径中带空格的文件。双引号可以使得`,`和`\`字符被正确读取。

通配符一定要小心使用，`*`会导致`logrotate`对所有文件进行切割转储，包含之前已经进行操作的文件（应该还包括切割出来的文件）。可以通过使用`olddir`指令，或者更精确的文件定义来解决。

### 常用配置

更多配置，可以参考[logrotate manual](https://linux.die.net/man/8/logrotate)，或者执行`man logrotate`查看。

* 周期

daily：指定操作周期为每天
weekly：指定操作周期为每周，当上次切割操作时间与现在间隔一周或以上时，执行。
monthly：指定操作周期为每月，当月第一次运行时，执行切割操作。
yearly：指定操作周期为每年，当上次切割操作时间与本次年份不一致时，执行。

* 压缩

compress：通过`gzip`压缩转储以后的日志
nocompress：不压缩
compresscmd：压缩时使用的命令，默认为 gzip
uncompresscmd：解压时命令，默认为 gunzip
compressext：指定开启压缩时，进行压缩的文件后缀名。默认跟随压缩工具支持的值。
compressoptions：执行压缩的参数，对于 gzip，默认参数为 -9（最大压缩比例）
delaycompress：将压缩时机推迟到下次滚动操作时。该操作仅在与`compress`联合使用才有效。常用在某些进程无法关闭日志文件输出位置而导致持续向上一个文件输出的情况。

* 常规

include：如果是文件夹，根据字母表顺序加载其中内容。（支持常规文件，排除文件夹以及部分不支持的文件，以及`taboo`指定排除的文件）。该指令，只能不能出现在文件级别的配置中。
missingok：忽略错误
tabooext `[+] list`：不执行的后缀列表，使用 + 代表，追加。默认不操作列表为：`.rpmorig, .rpmsave, ,v, .swp, .rpmnew, ~, .cfsaved .rhn-cfg-tmp-*`

* 存储

**minsize**：只有当文件超出规定大小时，才允许切割行为，需要同时满足指定的切割周期。`size`参数与其相似，但是它与时间周期参数互斥，它会导致切割行为不考虑上次切割时间（根据`size`参数的文档以及后续文字，推测这里的它是指`size`参数，而非`minsize`）。`minsize`参数会使得切割行为，需要同时考虑日志的大小和时间戳。
**size**：只有当文件超出指定大小时，才执行切割（有`crontab`执行`logrotate`的周期决定检查时间）。默认为`size`单位为 bytes，可以使用，K、M、G
olddir `directory`：切割文件都放入该目录中。路径需要是同一个物理设备上，除非是绝对路径，否则路径相对于存放日志文件的目录。可以被`noolddir`覆盖。
rotate：排除当前日志文件外的备份文件数。0 代表不进行备份。
dateext：使用日期格式，而不是简单的添加数字作为切割出来的日志名称。日期格式，可以使用`dateformat`定义。可以使用`nodateext`参数覆盖。
dateformat `format_string`：备份文件时，日期的格式。只允许使用`%Y``%m``%d``%s`四类格式符，默认格式`-%Y%m%d`。`%s`只有在 2001 年9月9日后，才能正常工作。日期格式，需要保证，较晚的日期，排序时，在后面，因为`logrotate`会根据文件名进行排序，然后删除旧的。
start `count`：切割文件命名数字时的起始值。
shred：使用`shred -u`处理需要删除的日志，而不是`unlink()`，这可以保证删除后的日志无法被读取。默认关闭，可以使用`noshred`参数覆盖。
* 脚本

postrotate/endscript：在这两个参数之间的内容，会在日志文件执行切割后，使用`/bin/sh`执行。一般只在针对特定文件进行定义，用来告知应用程序，已经发生日志切割操作。**只有当真正执行了切割操作，才会执行**。日志文件的绝对路径，会以第一个参数的形式，传入到对应脚本中。如果设置了`sharedscripts`，则整个匹配公式都会当做第一个参数传入。
prerotate/endscript：该参数之间的内容，会在文件日志**满足切割条件**，进行实际切割操作之前执行。其他行为类似`postrotate`。
firstaction/endscript：在通配符所匹配的所有文件名，执行切割之前，和`prerotate`之前执行。如果该脚本没有正常退出，后续操作不会执行。
lastaction/endscript：当`postrotate`执行后，和所有满足条件的日志文件切割后，才会执行。当该脚本没有正常退出时，只会输出一个错误日志。
sharedscripts：一般情况下，`prerotate`与`postrotate`会对每个匹配的日志分别执行，设置该行为后，以上脚本只会执行一次。如果没有任何文件需要被转储，那么这些脚本不会执行。

>一般如果是`create`的话，都需要执行一些命令，来通知进程。

### 手动执行

```shell
# 全局
logrotate /etc/logrotate.conf
# 自定义配置
logrotate /etc/logrotate.d/{{xx}}.conf
```

>推荐使用 -d 参数进行验证，而不实际操作。

## 参考资料：

1. [logrotate原理介绍和配置详解](https://wsgzao.github.io/post/logrotate/)
2. [logrotate manual](https://linux.die.net/man/8/logrotate)