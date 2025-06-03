> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287060.htm)

> [转帖]【银行逆向百例】01 小程序逆向之 webview 动态调试

**“** 这是一场试炼 **”**  

01

—

环境版本

环境：

手机，Android 15 未 root

电脑，Windows 11 专业版 23H2

软件：

微信，Android 8.0.58

edge，136.0.3240.50

02

—  

操作步骤

1、Android 手机开发者选项打开 USB 调试

![](https://bbs.kanxue.com/upload/attach/202506/1037623_RSGV98XKEDG9QAB.jpg)

2、微信开启调试

```
http://debugxweb.qq.com/?inspector=true
```

![](https://bbs.kanxue.com/upload/attach/202506/1037623_MYHHRBWZ72MWEDJ.jpg)

3、打开目标小程序，USB 连接电脑，默认仅充电即可

![](https://bbs.kanxue.com/upload/attach/202506/1037623_BZUXZH7N26PPNGW.jpg)

4、edge 浏览器打开

```
edge://inspect/#devices
```

![](https://bbs.kanxue.com/upload/attach/202506/1037623_J4RGHD4JXEW6MZ7.jpg)

5、手机允许 USB 调试

![](https://bbs.kanxue.com/upload/attach/202506/1037623_GMSE63GNZA2P773.jpg)

6、edge 调试资讯页面

![](https://bbs.kanxue.com/upload/attach/202506/1037623_6Q48CQUTSZ5PDMW.jpg)

7、搜索 encrypt 关键字，断点发现 he 为明文，pe 为 key，1 为加密，xe 为 undefined

![](https://bbs.kanxue.com/upload/attach/202506/1037623_EKE6VYHTNT4WDHK.jpg)

8、调用 sm4 函数与网络验证，输出结果与请求加密参数 requestData 一致

![](https://bbs.kanxue.com/upload/attach/202506/1037623_6HB52FRNRRGCMZK.jpg)

9、断点 decrypt，发现 he 为密文，pe 为 key，0 为解密，xe 为 undefined

![](https://bbs.kanxue.com/upload/attach/202506/1037623_NMG5HFS6BGVB9PE.jpg)

10、全局导出 sm4 函数，将网络的响应加密参数 responseData 赋值给 he，key 赋值给 pe

![](https://bbs.kanxue.com/upload/attach/202506/1037623_8SZXBAFCWWXD574.jpg)

11、调用 sm4 解密出明文响应

![](https://bbs.kanxue.com/upload/attach/202506/1037623_U32H8VQ5UCZZN75.jpg)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#其他](forum-151-1-212.htm)