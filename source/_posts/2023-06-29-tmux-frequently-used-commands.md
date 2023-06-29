---
title: Tmux 常用命令和快捷键
date: 2023-06-29 18:44:44
tags:
    - tmux
---

## Tmux 是什么
Tmux 是一个终端复用器（Terminal Multiplexer），它可以将终端的会话和窗口解绑分离，达到窗口关闭，会话不终止，并且还能使用窗口再次绑定目标会话的效果，从而避免因为网络中断或者窗口关闭导致会话中运行的进程终止。


## 基本用法
Ubuntu server 20.04 默认已安装 Tmux。如果没有，可以通过以下命令安装：
```shell
$ sudo apt-get install tmux
```

### 会话管理
#### 新建会话
```shell
$ tmux new -s <session-name>
```
#### 分离会话
```shell
$ tmux detach
```
#### 查看会话列表
```shell
$ tmux ls
```
#### 接入会话
```shell
$ tmux attach -t <session-name>
$ tmux attach -t <session-id>
```
#### 杀死会话
```shell
$ tmux kill-session -t <session-name>
$ tmux kill-session -t <session-id>
```
#### 切换会话
```shell
$ tmux switch -t <session-name>
$ tmux switch -t <session-id>
```
#### 重命名会话
```shell
$ tmux rename-session -t <old> <new>
$ tmux rename-session <new>
```
#### 会话快捷键
- **`Ctrl+b d`：分离当前会话，猜测 detach**
- **`Ctrl+b s`：查看会话列表并选择，session**
- **`Ctrl+b $`：重命名当前会话名称**
- **`Ctrl+d`：关闭当前会话**


### 窗口管理
#### 新建窗口
```shell
$ tmux new-window -n <window-name>
```
#### 关闭窗口
```shell
$ tmux kill-window -t <window-name>
$ tmux kill-window
```
#### 切换窗口
```shell
$ tmux select-window -t <window-name>
$ tmux select-window -t <window-id>
```
#### 重命名窗口
```shell
$ tmux rename-window -t <old> <new>
$ tmux rename-window <new>
```
#### 窗口快捷键
- **`Ctrl+b c`：创建一个新窗口，状态栏会显示窗口信息**
- `Ctrl+b p`：切换到上一个窗口，prev
- `Ctrl+b n`：切换到下一个窗口，next
- `Ctrl+b <number>`：切换到指定编号的窗口
- **`Ctrl+b w`：查看窗口列表并选择，window**
- **`Ctrl+b ,`：重命名当前窗口**
- **`Ctrl+b &`：关闭当前窗口**


### 窗格管理
#### 划分窗格
猜测 vertical，horizontal。
```shell
$ tmux split-window [-v]
$ tmux split-window -h
```
#### 移动光标
猜测 upper，down，left，right。
```shell
$ tmux select-pane -U
$ tmux select-pane -D
$ tmux select-pane -L
$ tmux select-pane -R
```
#### 交换窗格
当前窗格和上一个或下一个窗格交换位置。
```shell
$ tmux swap-pane -U
$ tmux swap-pane -D
```
#### 窗格快捷键
- **`Ctrl+b "`：上下划分窗格**
- **`Ctrl+b %`：左右划分窗格**
- **`Ctrl+b <arrow key>`：使用方向键切换窗格**
- `Ctrl+b ;`：光标切换到上一个窗格
- `Ctrl+b o`：光标切换到下一个窗格
- `Ctrl+b {`：当前窗格和上一个窗格交换位置
- `Ctrl+b }`：当前窗格和下一个窗格交换位置
- **`Ctrl+b x`：关闭当前窗格**
- `Ctrl+b !`：将当前窗格拆分为一个独立窗口
- `Ctrl+b z`：当前窗格全屏显示，再使用一次会变回原来大小，状态栏会有大写的 Z 标识
- `Ctrl+b q`：显示窗格编号


### 其他命令
#### 进入翻页模式
- **`Ctrl+b [`：进入翻页模式**
- `q`：退出翻页模式
#### 列出所有快捷键，及其对应的 Tmux 命令
```shell
$ tmux list-keys
```
#### 列出所有 Tmux 命令及其参数
```shell
$ tmux list-commands
```

## 参考链接
[Tmux 使用教程](https://www.ruanyifeng.com/blog/2019/10/tmux.html)
[Tmux 学习](https://blog.csdn.net/weixin_51311218/article/details/122172339)