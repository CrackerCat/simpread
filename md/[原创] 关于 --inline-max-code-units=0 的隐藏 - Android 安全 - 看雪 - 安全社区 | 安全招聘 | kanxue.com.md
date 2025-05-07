> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286498.htm)

> [原创] 关于 --inline-max-code-units=0 的隐藏

前言
--

在 LSPosed 中, 已通过`mount --bind`的方式拦截了`dex2oat`的调用, 并添加了`--inline-max-code-units=0`参数, 但是在`base.odex`中还是有痕迹, `Native Detector`还是可以检测出来.  
![](https://bbs.kanxue.com/upload/attach/202504/98159_3VCF67PV7FBU2ET.webp)  
LSPosed 的实现参考: [Xposed 框架 ART 参数修复](elink@c52K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6F1N6h3I4D9M7s2c8J5i4K6u0W2K9h3y4#2i4K6u0r3K9h3&6V1k6i4S2Q4x3X3g2H3K9s2m8Q4x3V1k6S2M7X3y4Z5K9i4k6W2M7#2)9J5c8U0f1K6i4K6u0r3)

确认
--

在 adb shell 中

```
cd /data/app/~~*==/com.reveny.nativecheck-*/oat/arm64/
strings base.odex | grep dex2oat-cmdline -A3
```

![](https://bbs.kanxue.com/upload/attach/202504/98159_DURFJTDD44Y75BX.webp)

解决
--

> 在生成`base.odex`后删除相关字符串

### 思路

由于`LSPosed daemon`已经拦截中真实的`dex2oat`, 直接在假的 dex2oat 与 daemon 通信时传递`--oat-location`中的路径和通知真实 dex2oat 运行结束就行

1.  传递 odex 路径
2.  `fexecve`会替换进程, fork 后, 等待结束后通知 daemon
3.  daemon 收到结束通知后 (要等一会, 文件好像没写入), 替换`base.odex`中的`--inline-max-code-units=0`为等长的空格

### 代码 (这里我偷个懒, 直接用 shell 执行替换)

`dex2oat/src/main/cpp/dex2oat.c`  
在`write_int`传递路径

```
write_int(sock_fd, ID_VEC(LP_SELECT(0, 1), strstr(argv[0], "dex2oatd") != NULL));
char location_odex[NAME_MAX];
for (int i = 1; i < argc; i++) {
    // LOGD("dex2oat argv[%d]: %s", i, argv[i]);
    if (strncmp(argv[i], "--oat-location=", 15) == 0) {
        strcpy(location_odex, argv[i] + 15);
        LOGV("dex2oat location-odex: %s, uid: %d", location_odex, getuid());
    }
}
size_t ps = strlen(location_odex);
write(sock_fd, &ps, sizeof(char)); // 255以下
write(sock_fd, location_odex, ps);
int stock_fd = recv_fd(sock_fd);
```

`daemon/src/main/java/org/lsposed/lspd/service/Dex2OatService.java`

```
var id = is.read();
var ignore = is.skip(3);
int path_len = is.read();
byte[] bytes = new byte[path_len];
if (is.read(bytes) > 0) {
    String odex = new String(bytes).trim();
    Credentials credentials = client.getPeerCredentials();
    new Thread(() -> waitDex2oatExit(odex, credentials)).start();
}
```

`waitDex2oatExit`

```
private void waitDex2oatExit(String odex, Credentials credentials) {
    try {
        Log.d(TAG, "path=" + odex + ", uid=" + Os.getuid() +
                   ", peer_uid=" + credentials.getUid() +
                   ", peer_pid=" + credentials.getPid() +
                   ", path_len=" + odex.length());
        String command = "while [ -d \"/proc/" + credentials.getPid() + "\" ]; do sleep 1; done";
        java.lang.Process process = Runtime.getRuntime().exec(new String[]{"/system/bin/sh", "-c", command});
        process.waitFor(1, TimeUnit.HOURS);
        // 要替换成等长字符 has invalid shdr offset/size: 123616/640
        String command2 = "sleep 3; PATH=$(magisk --path)/.magisk/busybox:$PATH; sed -i 's/--inline-max-code-units=0/                         /g' '" + odex + "'";
        java.lang.Process process2 = Runtime.getRuntime().exec(new String[]{"/system/bin/sh", "-c", command2});
        new Thread(() -> {
            try (var out = new BufferedReader(new InputStreamReader(process2.getInputStream()))) {
                String line;
                while ((line = out.readLine()) != null) {
                    Log.i(TAG, "dex2oat out: " + line);
                }
            } catch (IOException e) {
                Log.e(TAG, "Failed to read dex2oat output", e);
            }
        }).start();
        new Thread(() -> {
            try (var err = new BufferedReader(new InputStreamReader(process2.getErrorStream()))) {
                String line;
                while ((line = err.readLine()) != null) {
                    Log.e(TAG, "dex2oat err: " + line);
                }
            } catch (IOException e) {
                Log.e(TAG, "Failed to read dex2oat errput", e);
            }
        }).start();
        process2.waitFor(30, TimeUnit.SECONDS);
        Log.i(TAG, "dex2oat exit with code " + process2.exitValue());
    } catch (Exception e) {
        Log.e(TAG, "Failed to wait for dex2oat exit", e);
    }
}
```

确认修复
----

![](https://bbs.kanxue.com/upload/attach/202504/98159_J5MVCY4WUCQ2VFV.webp)

调试
--

`Native Detector`测试总是要卸载再安装, 可以使用`cmd package compile -m interpret-only -f com.reveny.nativecheck`主动调用 dex2oat

下一期讲讲怎么在 / proc/self/maps 中改名

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 2025-4-16 11:35 被无味编辑 ，原因： 代码错误

[#工具脚本](forum-161-1-128.htm)