# 安卓手机装 Linux

在 Android 设备上运行 Linux 主要有两种方式：

- 在 Android 里运行 Ubuntu 22.04（通过 Termux）：可行、安全、不需要解锁
- 彻底刷掉 Android，原生启动 Linux：多数 vivo 骁龙 888 机型目前基本不可行，除非已经有针对该型号的移植

本文以 `vivo x70 pro+`（骁龙 888+，12GB 内存，256GB 存储，Android 14）为例，介绍通过 Termux 运行 Ubuntu 22.04 的方法。

## 简介

[Termux](https://github.com/termux/termux-app) 是一个 Android 应用程序，它允许你在 Android 设备上运行 Linux 环境。它使用 APT 包管理器，可以安装各种 Linux 软件包，包括 Ubuntu 22.04。

## 安装 Termux

在 GitHub 下载 Termux 应用程序，并安装到你的 Android 设备上：

- https://github.com/termux/termux-app/releases
- 或镜像源：https://git.hiwiki.org/zh-CN/repo/termux/termux-app

打开 Termux 应用程序，并按照提示安装必要的软件包（最好挂上代理）：

```bash
pkg update -y
pkg upgrade -y
pkg install proot-distro -y
```

授权访问手机存储，弹出权限提示时选择允许：

```bash
termux-setup-storage
```

## 安装 Ubuntu 22.04

```bash
proot-distro install ubuntu:22.04 --name ubuntu2204
```

## 启动与退出

```bash
# 启动
proot-distro login ubuntu2204

# 退出
exit
```

## 配置环境

安装完后，进入 Ubuntu 22.04 环境，你可以使用 `apt` 命令来安装需要的软件包：

```bash
apt update
apt upgrade -y
apt install sudo nano git curl wget python3 build-essential
```

检查版本：

```bash
cat /etc/os-release
```

应显示类似：

```bash
NAME="Ubuntu"
VERSION="22.04.x LTS (Jammy Jellyfish)"
```

设置中国时区：

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
apt install -y tzdata locales
locale-gen zh_CN.UTF-8 en_US.UTF-8
```

## 访问手机存储

手机共享存储通常可以在 Ubuntu 中通过以下目录访问：

```bash
cd /sdcard
# 或
cd /storage/emulated/0
```
