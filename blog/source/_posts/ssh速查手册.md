---
title: SSH 速查手册
date: 2020-01-01
tags: Linux
---

使用`man ssh`查看`ssh`命令，使用`man ssh_config`查看`ssh_config`配置。

ssh 从三个地方分别获取配置内容：

1. 命令行参数
2. 用户设置文件(~/.ssh/config)
3. 系统设置文件(/etc/ssh/ssh_config)

每一处，都只会选取第一个匹配的值。（config 文件内 host 匹配一次后，就不在读取后续 host 规则）

## server 端添加 ssh 公钥

* 方法一

```bash
# ~/.ssh/id_rsa.pub 为公钥的本地地址
# 中间ssh那句，其实就是ssh登录命令。user 为远程host登录用户名，remote_host 为远程的host地址
cat ~/.ssh/id_rsa.pub | ssh user@remote_host "umask 077; mkdir -p .ssh ; cat >> .ssh/authorized_keys"
```

* 方法二

```bash
# MacOS
brew install ssh-copy-id
# ssh-copy-id [-h|-?|-f|-n] [-i [identity_file]] [-p port] [[-o <ssh -o options>] ...] [user@]hostname
ssh-copy-id -i ~/.ssh/id_rsa.pub [user@]remote-host
# 操作完成后，不需要配置 ssh config，也能够直接 ssh user@remote-host 完成添加
```

### 批量添加

使用`shell`的数组以及遍历，完成批量添加。

* 同步等待

```bash
array=(121.41.71.210 121.40.212.247 121.41.66.178 121.41.71.210 121.40.212.247 121.41.66.178 120.27.212.127 121.43.168.191 121.43.162.33 121.40.224.210 121.40.70.93 47.99.236.146 121.40.224.210 121.40.70.93 47.99.236.146)
# 不能用 ~ ，可能需要一点研究（也许能用$HOME)
pub="/Users/yleaf/.ssh/id_rsa.pub"

# 也可以写成for element in ${array[*]}
for element in ${array[@]}
do
# add ssh pub key
ssh-copy-id -i $pub root@$element
done
```

## keep-alive

参考资料：[ssh-keep-alive](https://einverne.github.io/post/2017/05/ssh-keep-alive.html)

* 服务器端主动

```bash
vim /etc/ssh/sshd_config
# 每隔 30s 送 keep-alive 包保持连接
ClientAliveInterval 30
# 发送 60 次无响应，则断开连接
ClientAliveCountMax 60
```

* 客户端主动

```bash
# 类似
vim /etc/ssh/ssh_config
# 或者
# vim ~/.ssh/config
ServerAliveInterval 30
ServerAliveCountMax 60
```

配置以上配置后，`ssh`每隔30s向`server`端`sshd`发送`keep-alive`包，如果发送60次，`server`无回应断开连接。`server`端配置类似。

## 验证私钥

* 使用 shell 返回码验证

```bash
ssh -q user@downhost exit
# 失败为非零返回
echo $?
255
# 成功时，返回 0
echo $?
0
```

>目前问题：没有更简单，可以用来在脚本方面使用的方法

```bash
ssh root@47.99.236.142 -T
# 失败会有返回，成功就一直不动了……
ssh: connect to host 47.99.236.142 port 22: Connection refused
```

* GitHub 以及 GitLab

```bash
ssh -T git@github.com
Hi leavesster! You have successfully authenticated, but GitHub does not provide shell access.
# 端口号
ssh -T git@corp.frdic.com -p 301
# 如果进行了ssh-add添加私钥操作
Welcome to GitLab, ye!
# 私钥对应失败
Permission denied (publickey).
```

## 指纹查询

```bash
# 不同的指纹方式
ssh-keygen -lf ~/.ssh/id_rsa.pub
ssh-keygen -E md5 -lf ~/.ssh/id_rsa.pub
```

* 私钥反查公钥

```
# 通过私钥，查询出公钥内容
ssh-keygen -y -f ~/.ssh/id_rsa
```

## GitHub 用户公钥

访问 `https://api.github.com/users/:username/keys` 将 :username 替换为用户名。

## 在脚本中使用 ssh，并执行命令

```bash
# ssh 成功后，在对应主机执行 command 
ssh host -tt command;
```

-tt 强制分配伪终端

### MacOS 下，异步执行多个命令

```zsh
osascript -e 'tell app "Terminal"
    do script "ssh host1 -tt \"wget https://example.com/init.sh && chmod 777 ./init.sh && ./init.sh\""
end tell'
osascript -e 'tell app "Terminal"
    do script "ssh host2 -tt \"wget https://example.com/init.sh && chmod 777 ./init.sh && ./init.sh\""
end tell'
osascript -e 'tell app "Terminal"
    do script "ssh host3 -tt \"wget https://example.com/init.sh && chmod 777 ./init.sh && ./init.sh\""
end tell'
```

## 查看登陆日志

```bash
# server 端
cd /var/log
less secure
```

## 更换端口，关闭密码

```bash
vim /etc/ssh/sshd_config

# 默认端口22，修改后需要使用`ssh -p 322 name@remote_host`进行登陆
Port 322

# 禁止密码登陆
PasswordAuthentication no
# 响应认证变化？触发按键之类操作，redhat 默认关闭
ChallengeResponseAuthentication no
```

[SSH PasswordAuthentication vs ChallengeResponseAuthentication 14 Sep 2013](https://blog.tankywoo.com/linux/2013/09/14/ssh-passwordauthentication-vs-challengeresponseauthentication.html)

>配置完成后，需要重启 ssh 服务才能生效

## 重启 ssh 服务

```bash
#RHEL/CentOS系统
service sshd restart
#ubuntu系统
service ssh restart
#debian系统
/etc/init.d/ssh restart
```

## 代理

ssh 不走 shell 的环境变量代理，需要在`sshconfig`中自行配置。同时，通过`ssh`进行连接的工具，也需要在`ssh_config`中配置。

### git

`git clone`地址为`git@github.com`或者`ssh`协议的连接，都是走`ssh`的。

### scp

`scp source target`
`target`可以使用`ssh_config`的`Host`配置。

## 远程编辑文件

来源：[SSH通过跳板机连接服务器
](https://morland96.github.io/2017/10/31/ssh-jump/)

```ssh
vim scp://<remote-host>/<file-path>
```

scp会将文件拷贝至tmp文件处，然后由`vim`在本地编辑。
不过目前看下来，反应比较慢。

## 端口转发

>内容未验证

[SSH通过跳板机连接服务器](https://morland96.github.io/2017/10/31/ssh-jump/)

* 端口转发命令

```shell
ssh [-L address] [user@]hostname [command]
 -L [bind_address:]port:host:hostport
 -L [bind_address:]port:remote_socket
 -L local_socket:host:hostport
 -L local_socket:remote_socket

# 完整用法：
 ssh -L 2222:<remote-host>:22  user@<jump-server>
# ssh -L local-port:remote-address:remote-port user@ssh-host
```

将`remote-host`的22端口，通过`jump-server`转发到本地（运行这条命令的机器）的2222端口。所有对本地`2222`端口的请求，都会通过`jump-server`转发至`remote-host`的`22`端口。

>`[-R address]`参数和`[-L address]`刚好相反，等验证完成后，分别写一遍。

>尝试解释：
> `[-L address]`只是连接过程中的一个可选参数，`host:hostport`是由`remotehost`进行解析的，`port`是`client`端的 port。

## 跳板机

`client` -->> `jump-server` -->> `target-server`

通过`ssh_config`可以隐藏跳板机命令，大大简化执行时命令的复杂度。

### 1. 跳板机作为网络中转

[SSH_ProxyJump](https://wiki.gentoo.org/wiki/SSH_jump_host)

跳板机本身，不存任何内容，之所以需要跳板机，只是为了绕过网络连接的限制（target只开放内网访问）。

```shell
Host target
    User root
    IdentityFile ~/.ssh/target_rsa
    HostName yy.yy.yy.yy # jump 可以连接的 ip（未验证：是否支持 jump 可以解析的 host（client 不支持解析的）
    ProxyCommand ssh -W %h:%p jump
Host jump
    HostName xx.xx.xx.xx
    User root
    IdentityFile ~/.ssh/jump_rsa
```

* 总结

像往常一样配置两个服务器。需要跳板机的机器，额外加一个`ProxyCommand ssh -W %h:%p jump` 就可以。

* ssh 命令参数

`-W host:port`：将所有`client`上的标准输入和标准输出，转发至`host`的`port`端口。自动设置`-N, -T, ExitOnForwardFailure and ClearAllForwardings`四个选项，如果需要修改以上设置，可以通过`-o`，或者在`config`中配置。

#### ProxyJump 命令

proxyjump 能够实现相同效果。

```shell
Host target
    User root
    IdentityFile ~/.ssh/id_rsa
    HostName yy.yy.yy.yy
    ProxyJump jump
Host jump
    User app
    HostName xx.xx.xx.xx
    IdentityFile ~/.ssh/id_rsa
```

>注意，该功能，需要比较新的 ssh client 客户端支持。

### 2. 跳板机才有 target key

[ssh-from-a-through-b-to-c-using-private-key-on-b](https://serverfault.com/questions/337274/ssh-from-a-through-b-to-c-using-private-key-on-b)
[ssh-agent](https://wiki.archlinux.org/index.php/SSH_keys#SSH_agents)

```shell
# 需要先在本地启动 ssh-agent，有的已经默认启动。
# 目前测试下来，MacOS 不需要执行，Ubuntu18 需要。
eval $(ssh-agent)

# 可以根据以下命令的输出，判断是否已经开启 ssh-agent。
echo $SSH_AUTH_SOCK
```

```ssh_config
Host target
    User root
    HostName yy.yy.yy.yy
    ProxyCommand ssh -o 'ForwardAgent yes' jump 'ssh-add && nc %h %p'
Host jump
    User app
    HostName xx.xx.xx.xx
    IdentityFile ~/.ssh/id_rsa
```

>这种跳板模式，会把跳板机中的`ssh key`信息加入到当前机器的`ssh-agent`中，退出后仍然存在，会存在一定影响。

>可以使用`ssh-add -l`查看已添加私钥，使用`ssh-add -D`删除`ssh-agent`中所有私钥，使用`ssh-add -d`删除特定。

* 示例

```shell
ssh target
# 会有以下提示，是把 jump 的 私钥 key 给加进来了
Identity added: /home/app/.ssh/id_rsa (/home/app/.ssh/id_rsa)
```

# SSH Agent 

相关命令：`ssh-add`,`ssh-agent`

## Agent-forwarding 携带 ssh key 信息

[Using SSH agent forwarding](https://developer.github.com/v3/guides/using-ssh-agent-forwarding/)
[SSH-Agent-Forwarding](https://blog.csdn.net/mr_raptor/article/details/51779339)

在服务器上，使用本地 ssh key 进行操作。

```ssh
# config
Host example
    ForwardAgent yes
    
# or
ssh -A user@host    
```

完整操作
```shell
# 启动 ssh-agent macOS 不一定需要
eval $(ssh-agent)

# 使用 ssh-add 将当前机器账号私钥添加到 ssh-agent 中
# ssh-add 默认添加 ~/.ssh/id_rsa 系列默认私钥，如果要添加其他位置，需要指定地址
ssh-add

# 启用 agent-forwarding 连接
ssh -A example

# 然后可以进行一些，需要client私钥的行为，比如 GitHub clone
ssh -T git@github.com
```