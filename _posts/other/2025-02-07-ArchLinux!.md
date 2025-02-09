---
title: "ArchLinux!"
excerpt: "Use it!"
date: 2025-02-07T20:42:31+08:00
categories:
  - blog
tags:
  - archlinux

toc: true
toc_label: "目录"
toc_icon: "list"
---
很好，今天上班摸鱼看了会儿`opengl-soft`看上头了，又想尝试`vulkan`+`opengl_soft`+`ffmpeg`整一套跨平台组合拳。

所以我或许又会再次放下DX12的学习（虽然最近都没学了），开始整linux了。
又回到了大学的时候，的感觉。

所以我选择`ArchLinux`！极简主义，GUI就是累赘！种种原因我要把它装到u盘上，这样我就既可以在大屏幕上（好吧，目前做不到，因为不知道为什么，我上次尝试用我的笔记本连显示器还是失败了）用linux，还可以在平板上用，这样就可以舒舒服服的躺在床上了哈哈哈。

所以，准备俩u盘，一个作为安装介质一个作为安装目标。archlinux镜像，在[arhclinux.org][archlinux.org]上下载，可以选择国内的镜像网站。以及写入工具，我用的`rufus`。

好了！Linux！启动！！

---
# Install ArchLinux
现在已经进入安装系统了。经典的命令行模式。

大部分安装都是照着arch wiki的指南做的，我只说我需要的部分，如果有其他需求，请自行参考[Installation_guid][Installation_guid]。

对了，我还有个旧的arch的安装指南，也可以参考。[github install arch][github install arch]

## Set font
进去后先设置合适的字体
```
setfont ter-122b
setfont ter-124b 
setfont ter-128b
setfont ter-132b
```
虽然还有别的字体可以设置，但对我来说这就够了。要看别的自己参考wiki哦。

## Verify the boot mode
这我不管，我的都是UEFI的，BIOS的话自己参考wiki或别的教程喽。

## Connect to the internet
我用wifi，也可以手机差数据线供网。
用的`iwctl`。
```
iwctl   // 进入 iwctl
device list // 显示可用设备，如 wlan0
station wlan0 scan // 扫描wifi
station wlan0 get-networks // 显示wifi
station wlan0 connect "network name" // 连wifi，并输入密码
quit // 退出
ping archlinux.org // 看能不能访问网络
```
## Update the system clock
```
timedatectl
```
这里一般显示的时间是晚8个小时的，这是正常的。

## Partition the disk
终于到分盘了，我是单盘主义者。
还有，我是直接将`grub`也写到u盘的，并没有放到跟主机一样的启动系统里（以前搞坏过导致俩系统都嘎了，而且我要在不同电脑上都能用，所以引导程序也装u盘上）。至于双系统或BIOS启动的，参考别的教程或我的旧链接。
```
fdisk -l // 显示所有disk，这里字体设置太大了看不到所有的就放小字体
         // 或者通过插拔u盘确认，我的是/dev/sdc

fdisk /dev/sdc // 进入

g // 创建GPT分区

n // 创建新分区
Enter
Enter // 全默认
+512M // 分配大小
t // 改变分区类型
1 // 为EFI

n
Enter
Enter
Enter
t
Enter
23 // 改变为Linux Root (x86-64)

p // 查看

w // 确认无误后写入
```
> 嗯~~装linux使我心情愉悦
> 虽然想要充分发挥硬件性能（Gento，太难了，折腾不来），（Manjaro，自动适配驱动虽好，但图标太蠢啦）。我驱动也折腾不过来，反正用核显开发galengine也不是不行，就当是优化性能了。
> 虽然但是，最优解当然是有一台单独的电脑只装linux啦。

## Format the partitions
```
mkfs.fat -F 32 /sdv/sdc1
mkfs.ext4 /dev/sdc2
```
sdc1 和 sdc2 就是你刚创建的俩分区, efi和root。

要想整些花里胡巧的文件系统自己玩去。

## Mount the file systems
挂在分区，挂在完就可以安装程序辣
```
mount /dev/sdc2 /mnt
mount --mkdir /dev/sdc1 /mnt/boot
```
双系统自己玩去。

> cao!兴奋起来了。（喝咖啡喝的）

## Swapfile
交换文件，就是用来扩展内存的，当虚拟内存用。

也可以在分区时用swap分区，嘛，我没搞过。
```
dd if=/dev/zero of=/mnt/swapile bs=1M count=8192 status=progress // 我只有8G内存抱歉了啊，自己想分配多少改count去
chmod 0600 /mnt/swapfile
mkswap -U clear /mnt/swapfile
swapon /mnt/swapfile // 交换文件！启动！！
```
靠，这文件创的好慢啊。

## Select the mirrors
众所周知，没一点儿魔法的话，你基本是下不了国外的东西一点儿，所以linux所需要的组件你也下不了。正因如此，你才需要换源（除非你用软路由？）。
```
reflector -p https -c China --delay 3 --completion-percent 95 --sort score --save /etc/pacman.d/mirrorlist
```

## Install packages
来了来了
```
pacstrap -K /mnt base linux linux-firmware
```
想装别的花活自己玩去。

如果安装报错了的话尝试一下方式修复，然后再重新装。

还不行的话那就google去吧。（或者全部重来一遍也不是不行）
```
pacman-key --init
pacman-key --populate
pacman -Sy archlinux-keyring
```

## Fstab
fstab文件，用于给linux系统描述启动时需要挂载的文件系统的信息的。
```
genfstab -U /mnt >> /mnt/etc/fstab
```
我就不检查了，我先走了哈

## Chroot
切到挂载的root分区上
```
arch-chroot /mnt
```
很好，你现在就已经进入到你的linux系统里了。

## Time
设区时
```
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```

## Localization
本地化
```
pacman -S neovim terminus-font // 先装个neovim还有terminus-font
nvim /etc/locale.gen // 解注释掉你想要的语言，如en_US.UTF-8，当然要保存

locale-gen
nvim /etc/locale.conf
LANG=en_US.UTF-8 // 设置语言

nvim /etc/vconsole.conf
FONT=ter-128b   // 设置终端字体
```

## Network configuration
```
nvim /etc/hostname
arch    // 随便哪个你想设置的主机名字
```
里面还有写其他细节参考wiki或旧链接。
```
nvim /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.0.1   arch.localdomain  arch
```
设置网络，你可以选择其他的上网用的程序
```
pacman -S   networkmanager
systemctl enable NetworkManager.service
```

## Root password
```
passwd
```
设密码

## Boot program
```
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```
这里要装bios或双系统的就遭殃了，至少我以前搞得时候很麻烦，自己google或参考旧链接去。

## Reboot
ok，终于可以重启你的linux系统了
```
exit
swapoff /mnt/swapfile
umount -R /mnt
reboot
```

至此，如果你成功进入grub并能启动你的arch，恭喜你，安装完成。

哦对了，在你登录的时候，是只有root用户的，以及你设置的密码。至于创建你自己的用户请看下章。

哦对了，你可能需要关闭安全模式或一些其他的bios设置什么的来确保能正常启动。我只能说，google吧，或参考我的旧链接，虽然我不记得里面有没有写这部分。至少我的电脑以前搞过，可以正常启动，enmmm


很~~~好，我tmd的笔记本可以正常启动，surface go2却不行，nnd！靠，又想放弃了，白白浪费了一个晚上。

果然单机装linux才是王道啊。（啥时候再整一个）

---

很好，到周末了。我要重装。

我打算直接把surface装成linux整机，然后虚拟机跑win。
具体的guide我就不放了，看link。[Linux Surface][Linux Surface]



<!-- link -->
[archlinux.org]: https://archlinux.org
[Installation_guid]: https://wiki.archlinux.org/title/Installation_guide
[github install arch]: https://github.com/Myc123abc/arch
[Linux Surface]: https://github.com/linux-surface/linux-surface