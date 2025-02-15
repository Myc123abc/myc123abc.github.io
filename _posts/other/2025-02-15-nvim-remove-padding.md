---
title: nvim remove padding
excerpt: 在终端中打开nvim出现空隙的问题
date: 2025-2-15T20:09:57+08:00
cateories:
  - blog
tags:
  - nvim
---

啊哈，许久不见。距离上次debug ffplay已经过去一段时间了。现在忙着搞linux开发环境，等准备好后在继续debug。(我果然还是又要开始搞vulkan了嘛...)

在配置nvim时遇到了问题，终端中打开nvim，nvim周四周会出现空隙，这让视觉上看起来很不爽，所以调查了大概半天多，找到了适合我的解决情况，在这里记载以下。

首先感谢这个帖子，主要参考的它。[You can remove padding around Neovim instance with this one simple trick... ][remove padding in nvim]

---

具体导致这个问题的原因，看帖子去。
我直接说我的解决办法。

首先我用的nvim，插件管理器用的lazy，安装插件
`mini.nvim`。

这里我就直接把nvim的mini的配置贴出来了。
```lua
return {
  {
    'echasnovski/mini.nvim',
    version = false,
    config = function()
      require('mini.misc').setup_termbg_sync()
    end
  }
}
```
总之就是mini有个misc模块，misc有个功能`setup_termbg_sync`可以用来同步终端边框的颜色与nvim的颜色一致。

如果你的终端不是kitty启用透明度的话这样就可以解决了。但很不幸我是用的透明终端，仅用这个方式并不能解决我的问题，我打开后nvim整个也跟着终端变透明了，颜色甚至跟漂白了一样。并且这个插件也有个小bug，它有一定几率在nvim推出时并不能还原颜色，只是在我的电脑上是这样的。所以对我来说，我还需要这个。
```bash
#!/bin/bash

kitten @set-background-opacity 1

nvim "$@"

kitten @set-background-opacity 0.3

echo -ne "\033]111\007"
```
** 如果你并没有透明终端的问题，kitten的那几行你就可以删掉了。
{: .notice}

哦，对了，用kitty的透明度还要解决这个问题的话，记得要把`dynamic_background_opacity`选项打开，就是为了动态调整opacity。当然还要打开远程控制的选项。

好了，其实看起来很明了。就是在nvim打开之前关掉透明度，然后在退出之后重新打开就行。最后一行是用来关掉边框背景色同步的，为了确保上面说的概率性小bug不会出现，至少我现在用的听好的。

然后最后一步就是把nvim`定向`到这个脚本上，当然脚本要上执行权限可别忘了。
定向后以后你执行nvim就自动执行的这个脚本了。
```bash
# here is .bashrc
# after eidt, use 

alias nvim='/path/to/sh' # set your sh path
```
问题解决，mession complete!


<!--link-->
[remove padding in nvim]: https://www.reddit.com/r/neovim/comments/1ehidxy/you_can_remove_padding_around_neovim_instance/ 
