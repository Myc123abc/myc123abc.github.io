---
title: "ffmpeg Decode Audio"
excerpt: "ffmpeg音频解码"
date: 2025-01-31T9:20:00+08:00
categories:
  - blog
tags:
  - ffmpeg

toc: true
toc_label: "目录"
toc_icon: "list"
---

其实，说实话前面几章都没啥大用，对我来说这章才是重点。

这章主要是针对音频格式文件的解码，以mp3文件为例，其中也有不少坑就是了。(这也是我前两次学习ffmpeg失败的原因)

但就在昨天，终于解决了这个问题，然后今天我们来看一下。

**提示：** 目前我对于ffmpeg的学习主要就先到这里，毕竟当前的目的还只是利用它进行各种音频的解码（wav, ogg, acc, flac等我之后也会出一篇解码的文章吧）。然后就是把ffmpeg的解码功能做成一个库，并与xaudio2结合写一个简易的音频库。
{: .notice}

OK，这次的内容b站是参考不了一点儿（它甚至都没解码这一章，只有编码的部分。。。），所以我主要参考的还是官方文档的示例[decode_audio.c][decode_audio.c]

本章所用的源码，主要实现了mp3的解码，并运用在xaudio2上进行了一个简单的播放。
源码总共不到250行，但直接贴在这里还是太多了，要看源码的话请参考这里，[code][code]

___

# 简述

我先大概简述一下这个解码的过程吧。

1. 首先要打开mp3文件，并且要跳过ID3标头，这是重点，因为不跳过ID3标头直接用ffmpeg解码的话会报错的，之后我再细说。
2. 然后对ffmpeg的解码所用的组件进行初始化，有解码器和解析器。
3. 分别初始化一个ffmpeg的包和帧，用于解码过程中使用。并准备一个内存区域，或文件什么的，用于保存解码后的数据。
4. 开始解码。
5. ffmpeg解码完成，现在可以使用xaudio2进行播放了。（这部分内容其实不应该包含在这里，毕竟也不不属于解码的部分哈哈）

# 准备音频文件
我是直接用fopen打开的，（实际上用的是_wfopen_s，因为非s系列的不安全编译器编译不过去。w是因为读取的文件路径包含非ASCII码字符，用fopen的话会失败），然后就直接去跳过ID3 tag了（后面一些术语的描述我就直接用英语了，直接用中文还是太怪了哈哈）。

这里的ID3 tag，是mp3文件格式的一种非标准的容器格式，具体的内容自己去google吧。总之在处理mp3文件用ffmpeg时要手动跳过这个tag，ffmpeg不会自己跳过，因此导致在解码第一个packet时会找不到"正确的"（ffmpeg所认为的）mp3 packet从而导致`avcodec_send_packet`失败（这是解码时会用到的一个func）。

具体想要了解为什么的话可以参考下这个link[mp3 decoding using ffmpeg API (Header missing)][mp3 decoding using ffmpeg API (Header missing)]。

吐槽一下，这个回答的老哥他只说了现象但没说具体解决的方式，导致我当年学习ffmpeg时即便找到了这个帖子，也半天没能解决问题，遂放弃。只能说还是当初太年轻了哈哈哈。

还有要参考ID3 tag怎么跳过的话直接看上面提到的源码就行，之后也一样我就不重复了。

# 初始化decoder与parser
decoder，或者说codec（即编解码器，因为coder与decoder其实共用的一个struct `AVCodec`）。

decoder就是实际用于解码的玩意儿。用`avcodec_find_decoder`去生成它。这个函数只有一个参数，就是指定你要的decoder的类型，这里当然选的`AV_CODEC_ID_MP3`啦。还有就是，ffmpeg还提供了`avcodec_find_decoder_by_name`的func，就是通过名字来指定decoder，使用是直接把`AV_CODEC_ID_MP3`替换成`"mp3"`就行（注意，这里的mp3是字符串哦）。当然`"MP3"`估计也可以但我没试过哈哈。

然后是parser顾名思义这就是用来解析的，具体是解析什么，就是把原始数据，即压缩后的数据给它分成一段段的方便之后解码时将数据打成packet吧。应该是这样我也没细看哈哈。

parser的创建也是需要指定类型的，这里就直接用decoder的id数据来指定，调用的就是`av_parser_init(decoder->id)`。

哦对了，还有，decoder只进行初始化时不行的，你还需要干两件事，创建decoder解码用的上下文`AVCodecContext`以及打开decoder。

`AVCodecContext`，decoder context。context就是上下文，那这个context是什么呢，你把decoder想成一种工具，而context就是使用这个工具的工作台一样就行，就像opengl也有context来着？状态机那种，你用decoder进行的所有操作，它操作所生成的中间数据都是要保存在context中的，一些我们重点关注的就有`sample_rate`，`ch_layout`一些什么的，具体到就是当前所解码的packet它的音频的基本信息，采样率和通道数等等。（所以这也是我说为啥前面几章也不那么重要的原因之一啦，比如说你想知道的音频文件的基本信息就可以从这里获取，虽然也可以用avformat就是了）

decoder和decoderContext创建完后，就要去打开decoder了，这里是做了什么操作我也还不太清楚哈哈。这两个玩意儿用什么函数创建的参考代码去，我就不在这写了，不会再重复了哦。

# 准备解码
先是准备一个buffer用于存储解码后的数据，即pcm（google去）。然后是准备一个packet以及一个frame。

packet就是包，在ffmpeg中，packet就是直接从原始数据中取出来的，准备用于解码的数据。

frame就是帧，ffmpeg中frame用来存储解码后的数据，就是从packet中解码后的。

**注意：**这里有一点是我也没弄明白的，就是解码过程中使用的临时缓冲区（不是pcm的），为什么空间要预留RefillThresh的部分，即使我从代码中看这部分的空间并没有直接使用到。所以感觉是ffmpeg内部解码中会用到这部分空间，或者是其他特殊的音频格式需要有额外的空间进行其他操作吧。总之用ffmpeg解码时临时buffer就多预留这点儿就行，不过ffmpeg官方已经给过你标准的buffer大小了直接用就行哈哈。
{: .notice}

# 解码
终于到这部分了，也是最难的了吧。

解码这部分你怎么写都行，这里用的跟官方给的例子一样，先是直接读取一个buffer的数据，然后进入循环去处理。
```cpp
auto readSize = fread(buffer, 1, BufferSize, file);
while (readSize > 0)
// ...
```
然后循环处理的第一个部分就是`av_parser_parse2`，这个就是我前面说到的parser解析的数据部分，我给你们看下代码来说明一下。
```cpp
auto ret = av_parser_parse2(parser, decCtx, &pkt->data, &pkt->size,
                                    data, readSize,
                                    AV_NOPTS_VALUE, AV_NOPTS_VALUE, 0);
```
返回值就不用说了，查报错的。

parser就是使用的解析器。

decCtx就是AVCodecContext，用于辅助parser提供必要数据来进行解析的。

然后就是`&pkt->data`和`&pkt->size`了，这其实就相当于你读文件时，读出来的部分放到一个buffer里并且返回了读取到的size，就是这俩了。

然后是data和readSize，这俩就是你前面文件读取出来的buffer与read size（或者说readed size更准确些）。再说一遍就是你从mp3文件读取到的原始数据，即要解码的数据（parser会把它分成packet放到`pkt->data`里），readSize就是你读取到的原始数据的大小了。Over。

然后有一个部分。
```cpp
data += ret;
readSize -= ret;
```
第一个就是正常的指针移动没什么好说的，指定到当前已经解码完的位置。

第二个是什么嘞，readSize是你要解码的raw data的size，而你使用parser解析成packet时并不会一次性就把所有的readSize全都解析成一个packet，实际中你一个readSize包含不止一个packet的数据，甚至也不一定是整数个packet，比如二点五条哈哈，这也引入了之后一个处理我待会再说。

所以第二个`readSize -= ret`得到的结果，就是你当前解析的raw data剩余的还没解析的data size。OK。

再然后就是真正对packet的解码了，我待会儿再说。
```cpp
if (pkt->size) // 这个不需要我解释吧，就是pakcet有数据时才decode嘛
{
    decode(decCtx, pkt, frame, pcm);
}
```
接下来这个部分，就是说明刚刚`readSize -= ret`的用处。
```cpp
if (readSize < RefillThresh)
{
    memmove(buffer, data, readSize);
    data = buffer;
    auto len = fread(data + readSize, 1, BufferSize - readSize, file);
    if (len > 0)
    {
        readSize += len;
    }
}
```
你可以看到有`RefillThresh`这么个东西，这玩意儿是重新填充阈值（至少是直接这么翻译过来的哈）。

ffmpeg要求，为了保证解析的流畅性，即不会出现中途解析数据中断的情况，你要解析的raw data buffer中，至少要保证有`RefillThresh`这么多的数据（ffmpeg给的这个的数值是4096）。

所以当当前buffer的数据不到`RefillThresh`是，就要填充新的数据。

`memmove`就是将当前剩余未解析的数据移回到buffer开头，用于下次解析。（就像你没吃完的剩饭留着下次吃一样哈哈）

然后重置data指针回到开头，之后的`fread`就是从file中读取剩余的部分（BufferSize - readSize）到buffer，来保证带解析数据充足（其实之后些音频库的流式播放时也可以这样哈）

最后一个if判断就是读到的数据大小重新加入到readSize中，没有的话就代表没啥数据可读了，eof了（google去）。

哦对了还有点没提，`RefillThresh`除了保证解析数据充足外还有一点（甚至这点才是真正重要的），那就是防止剩余数据不满一包从而导致陷入死循环（虽然我没尝试过但我时这么认为的）。

前面说过readSize中的数据并不一定是整数个packet，这就导致了会有一种情况，即某一次解析时parser发现剩余数据不满一个packet返回ret为0，并且pkt->data与pkt->size中都为0，从而导致无效操作无效解码。然后你又没有检查阈值重新填充的操作，这就导致buffer中一直都剩余着不满一包的数据，parser又解析不了，于是就一直陷入死循环了。

等等，你说eof的情况会怎么样？对啊！我怎么没发现，看来我的推测还是有点儿问题的，debug一下去。

......

好吧，果然还是要debug一下的。当buffer中剩余数据不满一包时，parse后的ret返回仍为剩余的readSize（比如需要400bytes来解码一包，但readSize为100的情况），但pkt->data和pkt->size仍为0。所以没有阈值检测的话就直接跳出循环了而不是死循环，导致最终解析的数据不完整。

而阈值检查并填充数据保证解析一直进行，并在最后eof中填充完最后的数据，以至于parse可以解析到最后一个包从而退出。（parse就是说的`av_parser_parse2`哦）

最后，在整个循环结束后还有一段处理。
```cpp
pkt->data = nullptr;
pkt->size = 0;
decode(decCtx, pkt, frame, pcm);
```
这部分将pkt中的数据清零并重新解码的原因是，decCtx中可能会存有未解码的数据，所以需要将剩余的数据全部解码完。可能有点儿抽象，你把它想成cpp中的cout，cout所打印的数据并不会在调用它时直接输出，而是在buffer满了后或程序退出时才会全部输出，这就导致有时你认为它已经输出了，但实际cout的buffer中仍有剩余数据，所以在必要的时候需要cout flush来手动强制输出。这里的decCtx也是一样的，只不过它并不会在程序退出时自动解码剩余数据，所以你仍需要手动decode一下，并且packet的data与size要清零，以至于让ffmpeg知道，你要解码的是decCtx中的剩余数据。（这里的decCtx就是decoder Context应该不会有人不知道吧哈哈）

OK，至此，所有的mp3的原始数据都被解码到了pcm中，你可以后续利用这些数据做任何事情，比如在我的源码中最后用xaudio2进行了播放，（xaudio2的部分以后再说喽）。还有一点，播放的话你直接解码完整个文件再播放还是太慢了，在我的surface go2上（奔腾黄金处理器！）要等待25秒左右才能解码完然后才能听到音乐的播放。
所以一个很明显的改善就是流式播放 stream play。就是在你解析的中途一并播放，比如说解析三帧然后直接播放，边解析边播放保证播放流畅不会中断或杂音，并且在解析途中可以给解析后数据套上各种effect chain（虽然我还没实践过）进行各种渲染后再播放（比如可以实现淡入淡出什么的）。总之能玩的花样很多，容我再慢慢摸索。

好了，目前ffmpeg的部分就主要研究到这了，剩下的时间我先看看流式播放的，然后是解码其他音频格式文件的部分，最后再写一个音频库，重写一遍我之前用fmod写的音频播放器。
然后就要重新开始我的DX12的部分了。（目前要开始学lighting了！）

离galgame engine又近了一步哈哈！

<!-- link -->
[decode_audio.c]: https://www.ffmpeg.org/doxygen/trunk/decode_audio_8c-example.html
[code]: https://github.com/Myc123abc/learn-ffmpeg/blob/main/src/testDecode.cpp
[mp3 decoding using ffmpeg API (Header missing)]: https://stackoverflow.com/questions/9509284/mp3-decoding-using-ffmpeg-api-header-missing