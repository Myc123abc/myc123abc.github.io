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
```bash
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
PS1='\[\033[38;5;118m\]\W \$ \[\033[0m\]' # 超简单提示符
```

# network
```bash
nmcli device wifi
nmcli device wifi connect 'name' password 'password'
```
虽然networkmanager的操作界面看起来一点儿都不优雅，甚至还会污染提示符的颜色，而且密码也不是隐藏式输入。

但它简单易用。

** 吐槽一下，linux-surface提供的pacman安装surface驱动的方法目前已经不适用。当前可行的办法是手动下载包然后pacman -U本地安装。不过我尝试安装之后，键盘断触的问题还是没解决。啧啧啧，微软真就不适配linux呗
{: .notice}

好吧，解决了。安个gpm就能解决tty键盘断连的事儿。还得是大佬啊，如此敏锐的观察力。[Type Cover disconnects for a few seconds][keyboard disconnected]

# tlp
省电的
```bash
sudo pacman -S tlp
systemctl enable --now tlp.service
systemctl mask systemd-rfkill.service
systemctl mask systemd-rfkill.socket
```

# openssh
远程访问
```bash
sudo pacman -S openssh
sudo systemctl enable sshd
sudo systemctl start sshd
ip a # check ip

# on windows 
ssh usrname@ip
```

# hyprland
一款基于wayland的windows manager(准确来说好像并完全是wm，wm是基于x11的，而基于wayland的是compositor。管他呢，约定俗成wm就行了)。
```bash
sudo pacman -S hyprland kitty gtk3 # 不知道为什么，不安装gtk3 kitty起不起来，然鹅pacman并不会添加gtk3的依赖
```
config kitty
```bash
sudo pacman -S adobe-source-code-pro-fonts

```
config hyprland
```bash
# ~/.bash_profile
hyprland # add this to .bash_profile so that can be autostart after you login in tty

# ~/.config/hypr/hyprland.conf
# remove autogenerated=1
```
还有以下的pkg可以安装

# neofetch
```bash
sudo pacman -S neofetch imagemagick
# .config/neofetch/config.conf
image_backend="kitty"
image_source="/path/to/img"
```



[keyboard disconnected]: https://github.com/linux-surface/linux-surface/issues/342#issuecomment-1908496161
