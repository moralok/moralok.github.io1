---
title: Ubuntu server 20.04 安装后没有分配全部磁盘空间
date: 2023-06-25 00:04:43
tags: [linux, ubuntu]
---

使用 `VMware` 安装 `Ubuntu server 20.04`，注意到实际文件系统的总空间大小仅占设置的虚拟磁盘空间大小的一半左右。本文介绍了如何解决该问题。

<!-- more -->

> 最近在本地测试 `Kubesphere` 和 `Minikube`，使用 `Ubuntu server 20.04` 搭建了多个虚拟机，磁盘空间紧张。注意到在安装后，实际文件系统的总空间大小仅占设置的虚拟磁盘空间大小的一半左右。如果 `Ubuntu server 20.04` 安装时使用默认的 `LVM` 选项，就会出现这种情况。

## 解决步骤

1. 使用 `df -h` 命令显示文件系统的总空间和可用空间信息。分配了 `40G` 磁盘空间，可用仅 `19G`。
```shell
$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                               3.9G     0  3.9G   0% /dev
tmpfs                              792M  7.5M  785M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   19G   17G  995M  95% /
tmpfs                              3.9G     0  3.9G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda2                          2.0G  108M  1.7G   6% /boot
/dev/loop0                          64M   64M     0 100% /snap/core20/1828
/dev/loop2                          50M   50M     0 100% /snap/snapd/18357
/dev/loop1                          92M   92M     0 100% /snap/lxd/24061
tmpfs                              792M     0  792M   0% /run/user/1000
/dev/loop3                          54M   54M     0 100% /snap/snapd/19457
```
2. 使用 `sudo vgdisplay` 命令查看发现 `Free  PE / Size` 还有 `19G`。
```shell
$ sudo vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <38.00 GiB
  PE Size               4.00 MiB
  Total PE              9727
  Alloc PE / Size       4863 / <19.00 GiB
  Free  PE / Size       4864 / 19.00 GiB
  VG UUID               NuEjzH-CKXm-W6lA-gqzj-4bds-IR1Y-dTZ8IP
  ```
3. 使用 `sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv` 调整逻辑卷的大小。
```shell
$ sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
Size of logical volume ubuntu-vg/ubuntu-lv changed from <19.00 GiB (4863 extents) to <38.00 GiB (9727 extents).
Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```
4. 使用 `sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv` 调整文件系统的大小。
```shell
$ sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs 1.45.5 (07-Jan-2020)
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
old_desc_blocks = 3, new_desc_blocks = 5
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 9960448 (4k) blocks long.
```
5. 使用 `df -h` 命令再次查看，确认文件系统的总空间大小调整为 `38G`。
```shell
df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                               3.9G     0  3.9G   0% /dev
tmpfs                              792M  7.5M  785M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   38G   17G   19G  47% /
tmpfs                              3.9G     0  3.9G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda2                          2.0G  108M  1.7G   6% /boot
/dev/loop0                          64M   64M     0 100% /snap/core20/1828
/dev/loop2                          50M   50M     0 100% /snap/snapd/18357
/dev/loop1                          92M   92M     0 100% /snap/lxd/24061
tmpfs                              792M     0  792M   0% /run/user/1000
/dev/loop3                          54M   54M     0 100% /snap/snapd/19457
/dev/loop4                          64M   64M     0 100% /snap/core20/1950
```

## 参考链接
[ubuntu20.04 server 安装后磁盘空间只有一半的处理](https://blog.csdn.net/weixin_43302340/article/details/120341241)
[Ubuntu Server 20.04.1 LTS, not all disk space was allocated during installation?](https://askubuntu.com/questions/1269493/ubuntu-server-20-04-1-lts-not-all-disk-space-was-allocated-during-installation)