---
title: Linux 常用命令和快捷键
date: 2020-08-27 12:35:32
tags: [linux]
---

根据个人使用经验记录常用的 `Linux` 命令作为备忘清单。

<!-- more -->

## 帮助

> 个人感觉以下获取帮助信息的方式重要的地方之一在于可以查阅得到第一手的权威注释，对于难以记忆或者不了解的命令参数，帮助信息往往也并非简单明了到可以临时快速阅读，不如直接在网络上搜索再加以记录。推荐参阅 [Linux 命令大全|菜鸟教程](https://www.runoob.com/linux/linux-command-manual.html)。

|命令|描述|
|--|--|
|`man command`|显示命令的操作说明|
|`command --help`|显示命令的帮助信息|

## 文件相关

|命令|描述|
|--|--|
|`ls` </br> `ls -al`（`ll`）|list，列出文件信息|
|`cd`|change directory，切换工作目录|
|`pwd`|print working directory，显示当前工作目录|
|`touch`|更新访问和修改时间为当前时间，常用于创建空文件|
|`cat`|查看文件|
|`more`|查看文件，可翻页|
|`less`|查看文件，可翻页|
|`tail`|查看文件，末尾，可实时|
|`mkdir`|创建目录|
|`cp`|copy，复制文件|
|`mv`|move，重命名或移动文件|
|`rm` </br> `rm -rf /usr/local/go`（卸载go）|remove，删除文件|
|`find`|查找文件|
|`vim`|编辑文件|
|`nano`|编辑文件|

## 文件权限

> `r`、`w`、`x` 分别代表读（`read`）、写（`write`）、执行（`execute`）权限，分别对应值 `4`、`2`、`1`。
`u`、`g`、`o` 分别代表拥有者（`owner`）、所属群组（`group`）、其他人（`other`），`a` 代表全部。
`+`、`-` 分别代表新增和删除权限。

|命令|描述|
|--|--|
|`chown [-R] username /tmp/testfile`|change owner，修改文件的拥有者|
|`chgrp [-R] groupname /tmp/testfile`|change group，修改文件的所属群组|
|`chmod [-R] 644 /tmp/testfile` </br> `chmod [-R] u=rwx,go=rx /tmp/testfile`  </br> `chmod [-R] a-x /tmp/testfile`|change mode，修改文件的权限模式|

## 用户

|命令|描述|
|--|--|
|`su username` </br> `su -`（切换root）|switch user，切换用户|
|`adduser username sudo`（交互式，便捷） </br> `useradd username`|新增用户|
|`passwd username`|修改密码|
|`groups username`|查看用户所属的群组|
|`groupadd docker`|添加群组|
|`usermod -aG docker $USER`|将当前用户添加到群组|
|`newgrp docker`|切换群组登录|
|`cat /etc/passwd`|查看UID和账号的对应关系|
|`cat /etc/group`|查看GID和账号的对应关系|
|`cat /etc/shadow`|查看账号和密码的对应关系|
|`cat /etc/ssh/sshd_config`|查看sshd配置|

## 排查问题

|命令|描述|
|--|--|
|`top`|显示系统的整体性能信息以及正在运行的进程的相关信息|
|`ps -ef` </br> `ps -aux`|process status，显示当前进程的状态|
|`kill -s 9 pid`|删除执行中的程序或工作|
|`ifconfig`|显示网络设备信息|
|`netstat -a` </br> `netstat -tunlp \| grep port`|查看网络的联机状态|
|`du -h /tmp`|disk usage，显示目录或文件的大小|
|`df -h`|disk free，显示文件系统磁盘空间使用情况|
|`who`|显示当前在线用户|
|`last`|显示用户最近登录信息|

|命令|描述|
|--|--|
|`vim /etc/profile`|修改系统范围的配置|
|`vim $HOME/.bashrc`|修改用户范围的配置|
|`source /etc/profile`|重新加载配置文件|
|`export PATH=$PATH:/usr/local/go/bin`|导出变量|

## 数据流重定向

> 标准输入（`stdin`）：代码为 `0`，使用 `<` 或 `<<`；标准输出（`stdout`）：代码为 `1`，使用 `>` 或 `>>`；标准错误输出（`stderr`）：代码为 `2`，使用 `2>` 或 `2>>`。
`>` 覆盖写；`>>` 追加写。

|命令|描述|
|--|--|
|`ll / > /tmp/rootfile`||

## 管道命令

|命令|描述|
|--|--|
|`ls -al /etc \| less`||
|`echo ${PATH} \| cut -d ':' -f 5`||
|`export \| cut  -c 12-`||
|`last \| cut -d ' ' -f 1`||
|`grep [-acinv] [--color=auto] '查找字符' filename`||
|`ps-ef \| grep java`|列出当前所有进程|
|`sort [-fbMnrtuk] filename`||
|`uniq [-ic]`||
|`wc [-lwm]`||
|`tee [-a] file`||
|`split [--bl] file PREFIX`||

## 关机或重启

|命令|描述|
|--|--|
|`sync`|将数据同步写入硬盘|
|`shutdown`|关机|
|`shutdown -h now`|立刻关机|
|`shutdown -r -t 60`|60秒后重启|
|`shutdown -r now`|立即重启|
|`reboot`|重新启动|

## 其他

|命令|描述|
|--|--|
|`echo ${PATH}`|显示文本、变量|
|`date`|显示日期和时间|
|`cal [month] [year]`|显示日历|
|`locale`|显示支持的语系|
|`bc`|简单计算器，scale=小数位数|
|`tar` </br> `tar -zcvf test.tar.gz ./test`（压缩） </br> `tar -zxvf test.tar.gz`（解压）|归档|

|命令|描述|
|--|--|
|`systemctl daemon-reload`|重新加载systemd配置|
|`systemctl enable node_exporter`|enable一个或多个units文件|
|`systemctl disable node_exporter`|disable一个或多个units文件|
|`systemctl status node_exporter.service`|显示一个或多个units的运行时状态|
|`systemctl start node_exporter.service`|启动一个或多个units|
|`systemctl stop node_exporter.service`|停止一个或多个units|
|`systemctl restart node_exporter.service`|重启一个或多个units|

## 快捷键

|命令|描述|
|--|--|
|`[Tab]`|命令补全；文件补全；选项参数补全|
|`[Ctrl]-a`|移动光标到开头|
|`[Ctrl]-e`|移动光标到结尾|
|`[Ctrl]-c`|终止|
|`[Ctrl]-d`|退出当前窗口|
|`[Ctrl]-l`|清屏|
|`[Ctrl]-u`|剪切、删除光标之前的内容|
|`[Ctrl]-k`|剪切、删除光标之后的内容|
|`[Ctrl]-r`|查找最近使用的指令|