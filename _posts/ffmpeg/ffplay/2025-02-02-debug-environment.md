---
title: "ffplay debug environment"
excerpt: "ffplay调试环境搭建"
date: 2025-02-02T03:08:00+08:00
categories:
  - blog
tags:
  - ffmpeg
  - ffplay
---

凌晨半夜这会儿还挺激动的，自从开始学DX12后就很少用linux了，为了深入学习ffmpeg，学习ffplay代码，在windows的wsl上整个linux环境搞搞（直接在windows上调试ffplay半天没折腾过来，所以只好在linux上整，不过linux还是好啊，很适合开发环境（坏了，我又有想中途放弃DX12直接去学vulkan的感觉了））。

安装的话，wsl现在是wsl2也有linux内核了，直接当linux用就行。装的经典的Ubuntu（虽然更喜欢arch但在wsl上装arch还是有些折腾了，不如ubuntu来的稳定和快速些）。

进去后安装各种必要工具，make，gcc，pkg-config，gdb之类的，还要有`libsdl2-dev`，不然configure配置时ffplay因为没有SDL依赖导致不会被加入到编译中去。

然后是下载好ffmpeg源码，进去后配置configure。
```
./configure
    --enable-debug
    --disable-optimizations
    --extra-cflags="-g3"
    --extra-ldflags="-g3"
    --disable-stripping
    --prefix=/usr/local
    --enable-shared
```
配置完成后就可以开始make了，记得用`make -j8`开启多线程加速编译，毕竟这是个漫长的时间（虽然我的surface go2就俩核心，所以只能用到j4，j8只会适得其反了）。

编译好后可以安装也可以不安装，直接调试也行。安装的话`sudo make install`。

好了，现在你可以直接用gdb调试了，调试的程序是`ffplay_g`，这个就是debug版本。

当然我个人对直接用gdb调试不熟悉，nvim上用gdb调试也太久没用了，所以这里就用vscode调试了。
（vs好像也可以吧，我记得用linux开发环境的设置的）。

vscode的配置，你需要安装wsl的插件，以及c/c++的插件。
**注意：** 这个c/c++插件与你在windows的vscode上已经安装过的不一样，它是指定在wsl:Ubuntu上运行所安装的。不安装的话launch.json中type无法设置为cppdbg。
{: .notice--warning}
然后是`.vscode`中`launch.json`的配置。
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "gdb",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/ffplay_g",
            "args": [
                "test.mp3"
            ],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb"
        }

    ]
}
```
这里的`args`你按你传参的需求来，我是先测试音频的所以设的test.mp3。

`.vscode`我是直接放在ffmpeg源码目录下了，放其他地方或许也可以，但相应的你的`launch.json`的配置内容就要调整，`program`什么之类的。

啊，当然，你也可以调试别的。`ffmpeg_g`或`ffprobe_g`都可以。

配置设置完之后你就可以来到`ffplay.c`中打断点调试了。

`F5`！启动！