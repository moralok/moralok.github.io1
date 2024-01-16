---
title: Linux 常用命令和快捷键
date: 2020-08-27 12:35:32
tags: [linux]
---

根据个人使用经验记录常用的 `Linux` 命令作为备忘清单。

<!-- more -->

## 获取帮助信息

|命令|描述|
|--|--|
|`man command`|显示命令的操作说明|
|`command --help`|显示命令的帮助信息|

## 文件相关

|命令|描述|
|--|--|
|`ls` </br> `ls -al`|列出文件信息|
|`cd`|切换工作目录|
|`pwd`|显示当前工作目录|
|`touch`|更新访问和修改时间为当前时间，常用于创建空文件|
|`cat`|查看文件|
|`more`|查看文件，可翻页|
|`less`|查看文件，可翻页|
|`tail`|查看文件，末尾，可实时|
|`mkdir`|创建目录|
|`cp`|复制文件|
|`mv`|重命名或移动文件|
|`rm`|删除文件|
|`find`|查找文件|
|`vim`|编辑文件|
|`nano`|编辑文件|
|`tar`|归档|

## 文件权限

|命令|描述|
|--|--|
|`chgrp [-R] groupname /tmp/test`|修改文件所属用户组|
|`chown [-R] username /tmp/test`|修改文件拥有者|
|`chmod [-R] xyz /tmp/test` </br> `chmod 644 /tmp/test` </br> `chmod u=rwx,go=rx /tmp/test`  </br> `chmod a-x /tmp/test`|修改文件的权限|

|命令|描述|
|--|--|
|`su -`|切换 root 用户|

## 排查问题

|命令|描述|
|--|--|
|`ifconfig`|显示网络设备情况|
|`who`|查看目前在线用户|
|`netstat -a`|查看网络的联机状态|
|`netstat -tunlp \| grep 端口号`|查看网络的联机状态|
|`ps -ef`|列出当前所有进程|
|`ps -aux`|列出进程|
|`du -h /opt/test`|查看目录使用情况|
|`df -h`|查看磁盘空间使用情况|
|`top`|显示系统当前进程信息|
|`kill -s 9 27810`|杀死进程|
|`last`|查看登录记录|

## 数据流重定向

- 标准输入（stdin）：代码为 0，使用 < 或 <<
- 标准输出（stdout）：代码为 1，使用 > 或 >>
- 标准错误输出（stderr）：代码为 2，使用 2> 或 2>>

- `>` 覆盖写
- `>>` 追加写

|命令|描述|
|--|--|
|`ll / > ~/rootfile`||

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


## 其他

|命令|描述|
|--|--|
|`echo ${PATH}`|显示文本、变量|
|`date`|显示日期和时间|
|`cal [month] [year]`|显示日历|
|`locale`|显示支持的语系|
|`bc`|简单计算器，scale=小数位数|

## 关机或重启

|命令|描述|
|--|--|
|`sync`|将数据同步写入硬盘|
|`shutdown`|关机|
|`shutdown -h now`|立刻关机|
|`shutdown -r -t 60`|60秒后重启|
|`shutdown -r now`|立即重启|
|`reboot`|重新启动|

## 快捷键

|命令|描述|
|--|--|
|`[Tab]`|命令补全；文件补全；选项参数补全|
|`[Ctrl]-c`|终止|
|`[Ctrl]-d`|键盘输入结束|
|`[Shift]-{[Page Up]\|[Page Down]}`|翻页|