---
title: "After install arch"
excerpt: "安装完arch后要做的事"
date: 2025-02-09T22:18:00+08:00
categories:
  - blog
tags:
  - arch

toc: true
toc_label: "目录"
toc_icon: "list"
---
前情提要，在花费一整天（多好的一个周末啊呜呜呜）装了gentoo无果后，愤然不到半小时重装了arch，还得是arch。

于是或，装好arch后，当然要开始配置些别的玩意儿。

# sudo
1. 添加个人用户
2. 安装sudo
3. 切换用户
```
useradd -m -G wheel username
passwd username

pacman -S sudo
EDITOR=nvim visudo
delete # for %wheel ALL=(ALL:ALL) ALL

su - username   // change to user
su -            // change to root
```

# bashrc
```bash
nvim ~/.bashrc

PS1='\W \$ ' # 超简单提示符

export http_proxy=xxx # 设置网络代理
export https_proxy=xxx
```

# network
```bash
nmcli device wifi
nmcli device wifi connect 'name' password 'password'
```
虽然networkmanager的操作界面看起来一点儿都不优雅，甚至还会污染提示符的颜色，而且密码也不是隐藏式输入。

但它简单易用。

# tlp
省电的
```bash
sudo pacman -S tlp
systemctl enable --now tlp.service
systemctl mask systemd-rfkill.service
systemctl mask systemd-rfkill.socket
```