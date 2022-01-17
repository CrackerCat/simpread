> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271155.htm)

> [分享][分享]3 个重点，20 个函数分析，浅析 FFmpeg 转码过程

**写在前面**
--------

最近在做和转码有关的项目，接触到 ffmpeg 这个神器。从一开始简单的写脚本直接调用 ffmpeg 的可执行文件做些转码的工作，到后来需要写程序调用 ffmpeg 的 API。虽然上网搜了别人的 demo 稍微改改顺利完成了工作，但是对于 ffmpeg 这个黑盒子，还是有些好奇心和担心（项目中使用不了解的代码总是不那么放心），于是抽空翻了翻 ffmpeg 的源码，整理成文章给大家分享分享。

 

由于我并非做音频出身，对于音频一窍不通。ffmpeg 整个也非常庞大，所以这篇文章从 ffmpeg 提供的转码的 demo 开始，侧重于讲清楚整个输入 -> 转码 -> 输出的流程，并学习 ffmpeg 如何做到通用和可扩展性。

 

注：本文基于 ffmpeg 提供的 transcode_aac.c 样例。

**三个重点**
--------

转码的过程是怎么样的？简单来说就是从输入读取数据，解析原来的数据格式，转成目标数据格式，再将最终数据输出。这里就涉及到三个**点**：**数据输入和输出方式**，**数据的编码方式**及**数据的容器格式**（容器是用来区分不同文件的数据类型的，而编码格式则由音视频的压缩算法决定，一般所说的文件格式或者后缀名指的就是文件的容器。对于一种容器，可以包含不同编码格式的一种视频和音频）。

 

ffmpeg 是一个非常非常通用的工具，支持非常广的数据输入和输出，包括：hls 流，文件，内存等，支持各类数据编码格式，包括：aac，mp3 等等，同时支持多种容器格式，包括 ts，aac 等。另外 ffmpeg 是通过 C 语言实现的，如果是 C++，我们可以通过继承和多态来实现。定义一个 IO 的基类，一个 Format 的基类和一个 Codec 的基类，具体的输入输出协议继承 IO 基类实现各自的输入输出方法，具体的容器格式继承 Format 基类，具体的编码格式继承 Codec 基类。这篇文章也会简单讲解 ffmpeg 如何用 C 语言实现类似 C++ 的继承和多态。

**基本数据结构**
----------

ffmpeg 转码中最基本的结构为 AVFormatContext 和 AVCodecContext。AVCodecContext 负责编码，AVFormatContext 负责 IO 和容器格式。

 

我从 AVFormatContext 类抽离出三个基本的成员 iformat，oformat，pb。分别属于 AVInputFormat，AVOutputFormat，AVIOContext 类。iformat 为输入的数据格式，oformat 为输出的数据格式，pb 则负责输入输出。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_MWUYVCJTYVQE54C.jpg)

 

我把这三个类的定义抽离了出来简化了下，可以看出 AVInputFormat 声明了 read_packet 方法，AVOutputFormat 声明了 write_packet 方法，AVIOContext 声明了 read_packet, write_packet 方法。同时 AVInputFormat 和 AVOutputFormat 还有一个成员变量 name 用以标识该格式的后缀名。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_5URNGVX6FX4VXKE.jpg)

 

下一节我们会看到 Input/OutputForm 的 read/write packet 方法和 IOContext 的关系。

**输入函数调用图**
-----------

下面是初始化输入的整个过程的函数调用图。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_2KG9C54QGBX3Y9D.jpg)

 

首先从调用 open_input_file 开始，首先**解析输入的 protocol**。avio_open2 函数会调用一系列 helper 函数（ffurl_open，ffio_fdopen）分析输入的协议，设置 AVFormatContext 的 pb 变量的 read_packet 方法。而 av_probe_input_buffer2 函数则会**分析输入文件的格式**（从文件名解析或输入数据做判断），设置 AVFormatContext 的 iformat 的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_U55GTSJTUV5E4A9.jpg)

 

两个 read_packet 有什么关系呢？第二个函数调用图可以看出，iformat 的 read_packet 最终会调用 pb 的 read_packet 方法。意思就是**数据本身由 pb 的 read_packet 方法来读取，而 iformat 则会在输入的数据上做些格式相关的解析操作**（比如解析输入数据的头部，提取出输入数据中真正的音频 / 视频数据，再加以转码）。

**IO 相关代码**
-----------

直接看上面的图不太直观，这一节我把源码中各个步骤截图下来进行分析。

 

转码开始步骤，调用 open_input_file 函数，传入文件名。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_H2YVR75Y6Z63EQE.jpg)

 

avformat_open_input 函数会调用 init_input() 来处理输入文件。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_BVAYDS7AFE7Q6FS.jpg)

 

init_input 函数主要做两个事情，一是解析输入协议（如何读取数据？hls 流？文件？内存？），二是解析输入数据的格式（输入数据为 aac？ts？m4a？）

 

![](https://bbs.pediy.com/upload/attach/202201/944667_PDX8D59QWV52P3M.jpg)

 

avio_open2 函数首先调用 ffurl_open 函数，根据文件名来推断所属的输入协议（URLProtocol）。之后再调用 ffio_fdopen 设置 pb 的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_QKCU682AKYUMN9R.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_N7TJXMC34CCK9EB.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_FS5Q3FQRZJF5JYB.jpg)

 

上面几段代码的逻辑为：根据文件名查找对应的 URLProtocol-> 把该 URLProtocol 赋值给 URLContext 的 prot 成员变量 -> 创建 AVIOContext 实例，赋值给 AVFormatContext 的 pb 成员变量。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_KEGW7HWQRHNZPU4.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_B4G7FZU48TPDUMK.jpg)

 

这里设置了 AVIOContext 实例的 read_packet 为 ffurl_read 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_VX6Y3EZM5N5HGXF.jpg)

 

ffurl_read 方法其实就是调用 URLContext 的 prot（上面赋值的）的 url_read 方法。通过函数指针去调用具体的 URLContext 对象的 prot 成员变量的 url_read 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_B7QXEXHFH2KVWUH.jpg)

 

接下来看看解析输入数据格式的代码。av_probe_input_buffer2 函数调用 av_probe_input_format2 函数来推断数据数据的格式。从之前的图我们知道 * fmt 其实就是 & s->iformat。因此这里设置了 AVFormatContext 的 iformat 成员变量。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_7XUCU5WJHHTBVUW.jpg)

 

至此 AVFormatContext 对象的 iformat 和 pb 成员变量就设置好了。接下来看看如何读取输入开始转码。

 

av_read_frame 函数调用 read_frame_internal 函数开始读取数据。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_V8K2Z43PNGS2Y3N.jpg)

 

read_frame_internal 会调用 ff_read_packet，后者最终调用的是 iformat 成员变量的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_9HA94FG2DYJGKUW.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_YTVGNN39SZ4D5VU.jpg)

 

拿 aac 举例，aac 的 read_packet 方法实际上是 ff_raw_read_partial_packet 函数。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_JYHK43E8AX6G7UN.jpg)

 

ff_raw_read_partial_packet 会调用 ffio_read_partial，后者最终调用的是 AVFormatContext 的 pb 成员变量的 read_packet 方法。而我们知道 pb 成员的 read_packet 其实就是 ffurl_read，也就是具体输入 URLProtocl 的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_HVA3XYSRQ3JJH33.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_7EFPVETMEJHRAKB.jpg)

 

至此已经走完了整个输入的流程，输出也是类似的代码，这里就不再赘述。

**转码函数调用图**
-----------

上面关于 IO 的介绍我从输入的角度进行分析。接下来的转码过程我则从输出的角度进行分析。下图是转码过程的函数调用图（做了简化）。load_encode_and_write 调用 encode_audio_frame, encode_audio_frame 调用 avcodec_encode_audio2 来做实际的编码工作，最后调用 av_write_frame 将编码完的数据写入输出。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_QTW5UJHN5HN7KMD.jpg)

**转码相关代码**
----------

首先需要设置输出目标编码格式，下面的代码为设置编码格式（aac）的片段：

 

![](https://bbs.pediy.com/upload/attach/202201/944667_G9B4ZT955VDANH2.jpg)

 

在这里设置了 output_codec_context（AVCodecContext 类对象）之后，从前面的函数调用图，我们知道是 avcodec_encode_audio2 函数执行的转码过程：

 

![](https://bbs.pediy.com/upload/attach/202201/944667_7NADQTURVJEP3HC.jpg)

 

这里看到调用了 avctx（AVCodecContext 类对象）的 codec（AVCodec 类对象）成员变量的 encode2 方法去做编码操作。

 

转码这里专业性比较强，我并没有细读，因此这里简单带过。

**总结**
------

可以看出 ffmpeg 大量使用函数指针来实现类似 C++ 的继承 / 多态的效果。并且 ffmpeg 具有非常好的扩展性。如果我需要自定义一个新的输入协议，只需要自己定义一个新的 URLProtocol 对象，实现 read_packet 方法即可。如果需要自定义一个新的容器格式，只需要定义一个新的 AVInputFormat 对象，实现 read_packet 方法即可。如果需要自定义一个新的编码格式，只需要定义一个新的 AVCodec 对象，实现 encode2 方法即可。真是非常赞的代码架构设计！

> **本文涉及的资料全部打包放到我 Github 仓：** [GitHub：2022 年，最新 ffmpeg 资料整理，项目（调试可用），命令手册，文章，编解码论文，视频讲解，面试题全套资料](https://link.zhihu.com/?target=https%3A//github.com/0voice/ffmpeg_develop_doc) **有需要的可以前去下载，或者觉得还不错，请给我 Star，感谢支持！**

**写在前面**
--------

最近在做和转码有关的项目，接触到 ffmpeg 这个神器。从一开始简单的写脚本直接调用 ffmpeg 的可执行文件做些转码的工作，到后来需要写程序调用 ffmpeg 的 API。虽然上网搜了别人的 demo 稍微改改顺利完成了工作，但是对于 ffmpeg 这个黑盒子，还是有些好奇心和担心（项目中使用不了解的代码总是不那么放心），于是抽空翻了翻 ffmpeg 的源码，整理成文章给大家分享分享。

 

由于我并非做音频出身，对于音频一窍不通。ffmpeg 整个也非常庞大，所以这篇文章从 ffmpeg 提供的转码的 demo 开始，侧重于讲清楚整个输入 -> 转码 -> 输出的流程，并学习 ffmpeg 如何做到通用和可扩展性。

 

注：本文基于 ffmpeg 提供的 transcode_aac.c 样例。

**三个重点**
--------

转码的过程是怎么样的？简单来说就是从输入读取数据，解析原来的数据格式，转成目标数据格式，再将最终数据输出。这里就涉及到三个**点**：**数据输入和输出方式**，**数据的编码方式**及**数据的容器格式**（容器是用来区分不同文件的数据类型的，而编码格式则由音视频的压缩算法决定，一般所说的文件格式或者后缀名指的就是文件的容器。对于一种容器，可以包含不同编码格式的一种视频和音频）。

 

ffmpeg 是一个非常非常通用的工具，支持非常广的数据输入和输出，包括：hls 流，文件，内存等，支持各类数据编码格式，包括：aac，mp3 等等，同时支持多种容器格式，包括 ts，aac 等。另外 ffmpeg 是通过 C 语言实现的，如果是 C++，我们可以通过继承和多态来实现。定义一个 IO 的基类，一个 Format 的基类和一个 Codec 的基类，具体的输入输出协议继承 IO 基类实现各自的输入输出方法，具体的容器格式继承 Format 基类，具体的编码格式继承 Codec 基类。这篇文章也会简单讲解 ffmpeg 如何用 C 语言实现类似 C++ 的继承和多态。

**基本数据结构**
----------

ffmpeg 转码中最基本的结构为 AVFormatContext 和 AVCodecContext。AVCodecContext 负责编码，AVFormatContext 负责 IO 和容器格式。

 

我从 AVFormatContext 类抽离出三个基本的成员 iformat，oformat，pb。分别属于 AVInputFormat，AVOutputFormat，AVIOContext 类。iformat 为输入的数据格式，oformat 为输出的数据格式，pb 则负责输入输出。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_MWUYVCJTYVQE54C.jpg)

 

我把这三个类的定义抽离了出来简化了下，可以看出 AVInputFormat 声明了 read_packet 方法，AVOutputFormat 声明了 write_packet 方法，AVIOContext 声明了 read_packet, write_packet 方法。同时 AVInputFormat 和 AVOutputFormat 还有一个成员变量 name 用以标识该格式的后缀名。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_5URNGVX6FX4VXKE.jpg)

 

下一节我们会看到 Input/OutputForm 的 read/write packet 方法和 IOContext 的关系。

**输入函数调用图**
-----------

下面是初始化输入的整个过程的函数调用图。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_2KG9C54QGBX3Y9D.jpg)

 

首先从调用 open_input_file 开始，首先**解析输入的 protocol**。avio_open2 函数会调用一系列 helper 函数（ffurl_open，ffio_fdopen）分析输入的协议，设置 AVFormatContext 的 pb 变量的 read_packet 方法。而 av_probe_input_buffer2 函数则会**分析输入文件的格式**（从文件名解析或输入数据做判断），设置 AVFormatContext 的 iformat 的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_U55GTSJTUV5E4A9.jpg)

 

两个 read_packet 有什么关系呢？第二个函数调用图可以看出，iformat 的 read_packet 最终会调用 pb 的 read_packet 方法。意思就是**数据本身由 pb 的 read_packet 方法来读取，而 iformat 则会在输入的数据上做些格式相关的解析操作**（比如解析输入数据的头部，提取出输入数据中真正的音频 / 视频数据，再加以转码）。

**IO 相关代码**
-----------

直接看上面的图不太直观，这一节我把源码中各个步骤截图下来进行分析。

 

转码开始步骤，调用 open_input_file 函数，传入文件名。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_H2YVR75Y6Z63EQE.jpg)

 

avformat_open_input 函数会调用 init_input() 来处理输入文件。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_BVAYDS7AFE7Q6FS.jpg)

 

init_input 函数主要做两个事情，一是解析输入协议（如何读取数据？hls 流？文件？内存？），二是解析输入数据的格式（输入数据为 aac？ts？m4a？）

 

![](https://bbs.pediy.com/upload/attach/202201/944667_PDX8D59QWV52P3M.jpg)

 

avio_open2 函数首先调用 ffurl_open 函数，根据文件名来推断所属的输入协议（URLProtocol）。之后再调用 ffio_fdopen 设置 pb 的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_QKCU682AKYUMN9R.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_N7TJXMC34CCK9EB.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_FS5Q3FQRZJF5JYB.jpg)

 

上面几段代码的逻辑为：根据文件名查找对应的 URLProtocol-> 把该 URLProtocol 赋值给 URLContext 的 prot 成员变量 -> 创建 AVIOContext 实例，赋值给 AVFormatContext 的 pb 成员变量。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_KEGW7HWQRHNZPU4.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_B4G7FZU48TPDUMK.jpg)

 

这里设置了 AVIOContext 实例的 read_packet 为 ffurl_read 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_VX6Y3EZM5N5HGXF.jpg)

 

ffurl_read 方法其实就是调用 URLContext 的 prot（上面赋值的）的 url_read 方法。通过函数指针去调用具体的 URLContext 对象的 prot 成员变量的 url_read 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_B7QXEXHFH2KVWUH.jpg)

 

接下来看看解析输入数据格式的代码。av_probe_input_buffer2 函数调用 av_probe_input_format2 函数来推断数据数据的格式。从之前的图我们知道 * fmt 其实就是 & s->iformat。因此这里设置了 AVFormatContext 的 iformat 成员变量。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_7XUCU5WJHHTBVUW.jpg)

 

至此 AVFormatContext 对象的 iformat 和 pb 成员变量就设置好了。接下来看看如何读取输入开始转码。

 

av_read_frame 函数调用 read_frame_internal 函数开始读取数据。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_V8K2Z43PNGS2Y3N.jpg)

 

read_frame_internal 会调用 ff_read_packet，后者最终调用的是 iformat 成员变量的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_9HA94FG2DYJGKUW.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_YTVGNN39SZ4D5VU.jpg)

 

拿 aac 举例，aac 的 read_packet 方法实际上是 ff_raw_read_partial_packet 函数。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_JYHK43E8AX6G7UN.jpg)

 

ff_raw_read_partial_packet 会调用 ffio_read_partial，后者最终调用的是 AVFormatContext 的 pb 成员变量的 read_packet 方法。而我们知道 pb 成员的 read_packet 其实就是 ffurl_read，也就是具体输入 URLProtocl 的 read_packet 方法。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_HVA3XYSRQ3JJH33.jpg)

 

![](https://bbs.pediy.com/upload/attach/202201/944667_7EFPVETMEJHRAKB.jpg)

 

至此已经走完了整个输入的流程，输出也是类似的代码，这里就不再赘述。

**转码函数调用图**
-----------

上面关于 IO 的介绍我从输入的角度进行分析。接下来的转码过程我则从输出的角度进行分析。下图是转码过程的函数调用图（做了简化）。load_encode_and_write 调用 encode_audio_frame, encode_audio_frame 调用 avcodec_encode_audio2 来做实际的编码工作，最后调用 av_write_frame 将编码完的数据写入输出。

 

![](https://bbs.pediy.com/upload/attach/202201/944667_QTW5UJHN5HN7KMD.jpg)

**转码相关代码**
----------

首先需要设置输出目标编码格式，下面的代码为设置编码格式（aac）的片段：

 

![](https://bbs.pediy.com/upload/attach/202201/944667_G9B4ZT955VDANH2.jpg)

 

在这里设置了 output_codec_context（AVCodecContext 类对象）之后，从前面的函数调用图，我们知道是 avcodec_encode_audio2 函数执行的转码过程：

 

![](https://bbs.pediy.com/upload/attach/202201/944667_7NADQTURVJEP3HC.jpg)

 

这里看到调用了 avctx（AVCodecContext 类对象）的 codec（AVCodec 类对象）成员变量的 encode2 方法去做编码操作。

 

转码这里专业性比较强，我并没有细读，因此这里简单带过。

**总结**
------

可以看出 ffmpeg 大量使用函数指针来实现类似 C++ 的继承 / 多态的效果。并且 ffmpeg 具有非常好的扩展性。如果我需要自定义一个新的输入协议，只需要自己定义一个新的 URLProtocol 对象，实现 read_packet 方法即可。如果需要自定义一个新的容器格式，只需要定义一个新的 AVInputFormat 对象，实现 read_packet 方法即可。如果需要自定义一个新的编码格式，只需要定义一个新的 AVCodec 对象，实现 encode2 方法即可。真是非常赞的代码架构设计！

> **本文涉及的资料全部打包放到我 Github 仓：**  
> [GitHub：2022 年，最新 ffmpeg 资料整理，项目（调试可用），命令手册，文章，编解码论文，视频讲解，面试题全套资料](https://link.zhihu.com/?target=https%3A//github.com/0voice/ffmpeg_develop_doc)  
> **有需要的可以前去下载，或者觉得还不错，请给我 Star，感谢支持！**

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#基础知识](forum-41-1-130.htm)