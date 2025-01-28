---
title: "ffmpeg File Operation"
excerpt: "ffmpeg中文件操作"
date: 2025-01-28T22:14:00+08:00
categories:
  - blog
tags:
  - ffmpeg
---

很好，没想到这么快就踩坑了。待会儿我会提到。

还是直接上代码来看，先说文件操作。

**注意：** 代码展示处我就不重复放include的了，如果有变动的话我会标出来
{: .notice}
```cpp
    auto path = "D:\\music\\四季ノ唄.mp3";

    AVIOContext* ctx;
    int ret;

    ret = avio_open(&ctx, path, AVIO_FLAG_READ);
    if (ret < 0)
    {
        av_log(nullptr, AV_LOG_ERROR, "Can't open %s: %s\n", path, av_err2str(ret));
        exit(EXIT_FAILURE);
    }

    ret = avio_close(ctx);
    if (ret < 0)
    {
        av_log(nullptr, AV_LOG_ERROR, "Can't close %s: %s\n", path, av_err2str(ret));
        exit(EXIT_FAILURE);
    }
```
`AVIOContext`就是读取文件成功后所获得的信息。

`avio_open`用于打开文件，第一个参数就是用于接收结果的ctx，第二个是你要打开的文件，第三个是你打开的方式，`AVIO_FLAG_READ`是只读，同样还有只写和读写，具体的信息请参考ffmpeg doc(我有在log那篇提过哦)。

`avio_close`就是关闭你打开了的文件的内容，为了避免内存泄漏。

中间使用的`av_log`就是上一篇的内容就不多说了。

然后是目录的操作。
```cpp
    auto dir = "D:\\music";

    AVIODirContext* dirCtx;
    ret = avio_open_dir(&dirCtx, dir, nullptr);
    if (ret < 0)
    {
        av_log(nullptr, AV_LOG_ERROR, "Can't open %s: %s\n", dir, av_err2str(ret));
        exit(EXIT_FAILURE);
    }
```
`AVIODirContext`是用来接受打开的目录的信息。

`avio_open_dir`用于打开目录，前两个参数与`avio_open`一样就不多说了，第三个参数目前也用不到，是一个option，这里直接传空就行了。

然后，如果你是在windows下运行这段代码的，那恭喜你，你或许会遇到和我一样的问题。

程序输出结果会如下所示
```
Can't open D:\music: Function not implemented
```
没错，它报错了。

debug中会发现返回值为-40，调用`av_err2str`将错误码转化为字符串得到的结果就是`Function not implemented`。

这并不是因为你传入的dir有问题还是你用`avio_open_dir`的使用方式错了。

而是因为你在windows平台下使用了ffmpeg的`avio_open_dir`这个函数。

我在网上找了一番也没能找到这个问题的解答，于是自己在stackoverflow上问了一下，具体的话可以参考[FFmpeg avio_open_dir returns -40 on Windows, even when directory exists][question]

我简单解释一下就是`avio_open_dir`在打开本地目录时使用的`file_open_dir`，而`file_open_dir`是直接使用系统io去打开的，直接看7.1版本[源码][source]的话
```c
static int file_open_dir(URLContext *h)
{
#if HAVE_LSTAT
    FileContext *c = h->priv_data;

    c->dir = opendir(h->filename);
    if (!c->dir)
        return AVERROR(errno);

    return 0;
#else
    return AVERROR(ENOSYS);
#endif /* HAVE_LSTAT */
}
```
可以发现`file_open_dir`使用了`HAVE_LSTAT`这个宏。

有它的时候就调用`opendir`来打开目录，这是个linux下的io。

而没有这个宏是就直接返回报错了。
没错，file_open_dir竟然没有windows侧的处理，我当时也很震惊啊！

所以`avio_open_dir`这个函数，很遗憾，你在windows下是没办法直接调用了。

那解决办法是什么嘞，除非你想自己修改源码编译后再使用，否则还是老老实实用其他api打开文件吧。

windows的api或者c的或cpp的filesystem，这里我就先使用cpp的了，毕竟最熟悉嘛。

至于使用其他api如何像`avio_open_dir`获得的`AVIODirContext`一样使用在ffmpeg中去，我也不造，嘛，毕竟也还在学习嘛，走一步是一步喽。之后有可以补充的话我还会写在后头的，就先不操心啦。

**注意：** OK，还有点没说，b站的那个课程用的`avpriv_io_delete`与`avpriv_io_move`在ffmpeg 7.1版本中是废弃的。不过这俩也只是对文件的基本操作同样可以用其他api替代就是了。
{: .notice}

<!-- link -->
[question]: https://stackoverflow.com/questions/79393752/ffmpeg-avio-open-dir-returns-40-on-windows-even-when-directory-exists
[source]: https://github.com/FFmpeg/FFmpeg/blob/85a327d9d06a26c7743f4b14902b848dab42c44f/libavformat/file.c#L324-L337