> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [5ec1cff.github.io](https://5ec1cff.github.io/my-blog/2023/11/23/system-certs/)

> Android 安装系统证书抓包的时候通常需要安装特定的 CA 证书以便中间人解密，本文以安装 mitmproxy 证书为例介绍如何准备证书文件和临时安装（挂载）证书到特定的 app 。

发表于 2023-11-23|更新于 2023-11-23

| 阅读量:

抓包的时候通常需要安装特定的 CA 证书以便中间人解密，本文以安装 [mitmproxy](https://mitmproxy.org/) 证书为例介绍如何准备证书文件和临时安装（挂载）证书到特定的 app 。

[](#准备证书 "准备证书")准备证书
--------------------

这里可以直接参考 mitmproxy 给出的流程

[System CA on Android Emulator](https://docs.mitmproxy.org/stable/howto-install-system-trusted-ca-android/)

mitmproxy CA 证书位于 `~/.mitmproxy/mitmproxy-ca-cert.cer` (Windows: `%UserProfile%\.mitmproxy\mitmproxy-ca-cert.cer`) ，Android 接受这种格式的证书，不过需要重命名为特定的名字。

命令 `openssl x509 -inform PEM -subject_hash_old -in mitmproxy-ca-cert.cer` 第一行输出证书的 hash ，本例中为 `c8750f0d` ，因此复制一份 mitmproxy-ca-cert.cer ，将其重命名为 `c8750f0d.0` ，即可得到 Android 上可用的证书。

[](#挂载到系统目录 "挂载到系统目录")挂载到系统目录
-----------------------------

对于 Android 13 和之前的版本，系统默认读取 `/system/etc/security/cacerts` 下的 CA 证书。对于 Android 14 ，由于采用了 apex 实现可更新的 CA 证书，因此默认从 `/apex/com.android.conscrypt/cacerts` 读取。

如果需要挂载全局有效的证书，前者可以使用 Magisk 模块系统直接挂载，后者由于 Magisk 未实现 /apex 的覆盖，因此可以手动使用 mount 命令挂载。可以参考 AdGuard 的模块 adguardcert。

需要注意，如果不用模块挂载而是在系统启动后挂载证书，则 Android 13 及之前版本应该在全局 ns 挂载，在 Android 14 上要特别注意，因为 apex 目录不进行挂载传播，因此在全局挂载不会影响到 app 进程，除非重启 zygote 。建议的方法是挂载到 zygote ns （这样只需重启目标 app）或者直接挂载到 app ns

[https://github.com/AdguardTeam/adguardcert/releases/tag/v2.0](https://github.com/AdguardTeam/adguardcert/releases/tag/v2.0)

下面介绍针对特定 app 单独挂载。我们可以用一个 overlayfs 增加证书，并挂载到应用的 ns 。sh 脚本示范：

> 当然也可以像 adguard 一样模仿 magisk 构建 tmpfs

```
mkdir -p /mnt/cert
cp mitmproxy/c8750f0d.0 /mnt/cert
/system/bin/chcon u:object_r:system_security_cacerts_file:s0 /mnt/cert
chmod 755 /mnt/cert
/system/bin/chcon u:object_r:system_security_cacerts_file:s0 /mnt/cert/*
chmod 644 /mnt/cert/*

SYSTEM_CERTS_PATH=/system/etc/security/cacerts



pids=$(pgrep -f "$1")
for pid in $pids; do
echo "mount for $pid $(cat /proc/$pid/cmdline)"
nsenter --mount="/proc/$pid/ns/mnt" mount -t overlay CERTS -olowerdir=/mnt/cert:$SYSTEM_CERTS_PATH $SYSTEM_CERTS_PATH
done
```

当需要对特定 app 挂载证书的时候，启动 app 并执行 `./mount-cert.sh '$packageName.*'` 即可挂载证书到 app 的所有进程。

[](#源代码 "源代码")源代码
-----------------

读取证书的流程如下：

```
at android.security.net.config.DirectoryCertificateSource.readCertificate(DirectoryCertificateSource.java:227)
at android.security.net.config.DirectoryCertificateSource.findCerts(DirectoryCertificateSource.java:150)
at android.security.net.config.DirectoryCertificateSource.findAllByIssuerAndSignature(DirectoryCertificateSource.java:115)
at android.security.net.config.SystemCertificateSource.findAllByIssuerAndSignature(SystemCertificateSource.java:28)
at android.security.net.config.CertificatesEntryRef.findAllCertificatesByIssuerAndSignature(CertificatesEntryRef.java:65)
at android.security.net.config.NetworkSecurityConfig.findAllCertificatesByIssuerAndSignature(NetworkSecurityConfig.java:146)
at android.security.net.config.TrustedCertificateStoreAdapter.findAllIssuers(TrustedCertificateStoreAdapter.java:46)
at com.android.org.conscrypt.TrustManagerImpl.findAllTrustAnchorsByIssuerAndSignature(TrustManagerImpl.java:942)
at com.android.org.conscrypt.TrustManagerImpl.checkTrustedRecursive(TrustManagerImpl.java:558)
```

[frameworks/base/core/java/android/security/net/config/SystemCertificateSource.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/security/net/config/SystemCertificateSource.java;l=59;drc=9a8d2d1a3c349a14929b6a581ac0a8ed9b4b8364)