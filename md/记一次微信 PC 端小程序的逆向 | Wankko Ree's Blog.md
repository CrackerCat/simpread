> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.wkr.moe](https://www.wkr.moe/study/494.html)

> 前提说明 PC 端小程序的包有加密，需要解密后才能解包。

PC 端小程序的包有加密，需要解密后才能解包。

*   Windows`10.0.19042.928`
*   微信`3.2.1.132`
*   [pc_wxapkg_decrypt_python](https://github.com/kksanyu/pc_wxapkg_decrypt_python "pc_wxapkg_decrypt_python")`fdbe52f`
*   [mp-unpack](https://github.com/xuedingmiaojun/mp-unpack "mp-unpack")`1.1.1`

1.  目标小程序为`一起学党史`，图标如下  
    ![](https://i.loli.net/2021/04/14/BkXIoLwHeQGsf3W.png)
2.  进入`文档\WeChat Files`文件夹，找到你 wxid 的那个文件夹，如`wxid_abcdefghijkl21`，进入`文档\WeChat Files\你滴wxid\Applet`文件夹，找到小程序 wxid 的那个文件夹，记住小程序 wxid，如`wx6e24c691f9a50e44`，找法个人觉得有两种比较好：
    *   挨个进入`文档\WeChat Files\你滴wxid\Applet\小程序滴wxid\store\images`文件夹，看里面存的图片（无后缀，实际格式的话`png`和`jpg`都有），有你要找的小程序的贴图或者图标就是了
    *   删除除了`global`外的小程序文件夹，然后重新打开一下小程序，看看蹦出来的文件夹是哪个（没测试过行不行）
3.  进入`文档\WeChat Files\Applet\小程序滴wxid\可能是版本号`文件夹，拿到`__APP__.wxapkg`，可能还会有分包文件，本文不作讨论（主要是我没用上）
4.  命令行使用`pc_wxapkg_decrypt_python`解密小程序包  
    `python main.py --wxid 小程序滴wxid --file __APP__.wxapkg --output 解密后的文件名.wxapkg`
5.  使用`mp-unpack`解包小程序包

本文目标案例的小程序每个数据包都带了`sign`字段，长度 40，猜测可能是`sha1`哈希。

![](https://i.loli.net/2021/04/14/PtgWCONJhjluQdT.png)

查找文本发现如下可疑代码段。

![](https://i.loli.net/2021/04/14/KhFIJvnrX8jP5O3.png)

```
function getsign2(params) {
    var s_data = "";
    var keys = [];
    for (var key in params) {
        if (params.hasOwnProperty(key)) {
            var value = params[key];
            keys.push(key);
        }
    }
    var arr = [];
    keys.sort();
    for (var i = 0; i < keys.length; i++) {
        var k = keys[i];
        var v = params[k];
        arr.push(k + "=" + v);
    }
    s_data = arr.join("&");
    var sign = sha1(SparkMD5.hash(s_data) + "lanzu@123");
    params.sign = sign;
    return params;
}

```

直接控制台构造一个`params`进行模拟测试，看看是不是和抓到的数据包一样。

![](https://i.loli.net/2021/04/14/XoV3ihbRWGQjPuZ.png)

因为控制台不好导加密包，所以直接拿这一串去`sha1(md5(s_data)+'lanzu@123')`

得到结果发现与抓包结果一样。

![](https://i.loli.net/2021/04/14/AkYRr9gTDO7Ivjy.png)

直接 python 还原一下生成验签的步骤。

```
def sign(params: dict) -> dict:
    keys = list(params.keys())
    keys.sort()
    s_data = "&".join([f'{key}={params[key]}' for key in keys])
    sign = sha1(f'{md5(s_data.encode()).hexdigest()}lanzu@123'.encode()).hexdigest()
    params['sign'] = sign
    return params

```

* * *

The End