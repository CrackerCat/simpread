> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277869.htm)

> [原创] 入门级，无壳 app 登录算法还原并实现发包

前言
==

如题本文主要讲某 app 登录算法还原并实现发包的全过程，此 app 非常适合小白入门时拿来练手，  
一个无壳无混淆的 app，软件链接我会放在文章最下面，建议先看一下文章后，再去下载上手。

追踪关键代码
======

首先将 apk 拖入 jadx 进行关键字 LoginActivity 搜索，然后进行分析得到下图代码， ![](https://bbs.kanxue.com/upload/attach/202307/973686_PPH7STDSMM7NVGB.jpg)  
是手机号加密码的发包位置，其中可见手机号和密码都被传入了 login，跟踪进去后，可见的相关代码如下图。 ![](https://bbs.kanxue.com/upload/attach/202307/973686_CU92CNCDHZTRMPC.jpg)  
这里 login 又调用的私有方法 doLogin，将手机号和密码传给了 UserService 里面，我们进行追踪，得到如下图代码。  
![](https://bbs.kanxue.com/upload/attach/202307/973686_8DSXMRXSPPJVPBM.jpg)  
这里可以看到了密码登录参数，手机号和密码被传入了这里，密码被传进了 getMD5 后，加密后存入了 params 又传入了 getRSAParams，然后传入 request，先继续追踪 getMD5，  
![](https://bbs.kanxue.com/upload/attach/202307/973686_JPF7XRTS62TFZA8.jpg)  
是一个普通的 MD5，用 python 进行一下复现，代码如下。

部分代码复现：
-------

```
import hashlib
 
def get_md5(val):
    m = hashlib.md5()
    m.update(val.encode('utf-8'))
    result = m.hexdigest()
    return result
 
print(get_md5("qwerty"))
# 结果：d8578edf8458ce06fbc5bb76a58c5ca4

```

然后返回追进 getRSAParams 里面看到下图代码，一层 base64 加密，还有 RSA 加密。 ![](https://bbs.kanxue.com/upload/attach/202307/973686_FD3NP757BYZVK63.jpg)

上 frida，开始动刀子：
==============

先用 frida 来对这个对象动刀子，得结果如下

```
*getRSAParams is called, params: {password=d8578edf8458ce06fbc5bb76a58c5ca4, os=android, mobile=15536263522, version=2.2.3}
 
getRSAParams ret value is {data=eyJwYXNzd29yZCI6ImQ4NTc4ZWRmODQ1OGNlMDZmYmM1YmI3NmE1OGM1Y2E0Iiwib3MiOiJhbmRyb2lkIiwibW9iaWxlIjoiMTU1MzYyNjM1MjIiLCJ2ZXJzaW9uIjoiMi4yLjMifQ==,
 
sign=DmxjCCvf8aJnZNve4BQkcy6turIGzkE13DkIu9JSnJF7yUDp3ZUxANRnSn6+BCN2nEogZsHFOm0fzzTU/NMnEqijA8lklHoxZspzVOe6Hkp8jYRrzvf0PQIh25lEL2GGWSslgzEK710opNDoQUVHA95ArOv9FQN95HxZuj7ywio=,
 
timestamp=1687264102}*

```

传入之前的密码，python 复现的 MD5 结果相同，传入 getRSAParams 后成了后面的三项，data sign 和 timestamp，然后这些数据和 charles 抓包得到的 login 数据相同。

再次追踪关键代码
========

需要先解决 data，进行追踪得到下图代码。Base64 加密与解密。  
![](https://bbs.kanxue.com/upload/attach/202307/973686_WBMUZA8USA7JE96.jpg)  
对其进行 python 复现得到以下代码，运行结果与 frida 打印信息相同，

```
import base64
import json
 
def encode(data, charset='UTF-8'):
    datas=json.dumps(data,separators=(',', ':'))
    try:
       return base64.b64encode(datas.encode(charset)).decode(charset)
    except Exception as e:
        print('Error', e)
        return None
 
data = {"password":"d8578edf8458ce06fbc5bb76a58c5ca4","os":"android","mobile":"15536263522","version":"2.2.3"}
result = encode(data)
print(result)
#结果：eyJwYXNzd29yZCI6ImQ4NTc4ZWRmODQ1OGNlMDZmYmM1YmI3NmE1OGM1Y2E0Iiwib3MiOiJhbmRyb2lkIiwibW9iaWxlIjoiMTU1MzYyNjM1MjIiLCJ2ZXJzaW9uIjoiMi4yLjMifQ==

```

接着追踪 sign 的相关代码，得到下图代码 ![](https://bbs.kanxue.com/upload/attach/202307/973686_GVEGGPEW3FH489K.jpg)

frida 再次出刀
==========

这里用 frida 进行 hook，得到信息如下，可见这里是 data 与 timestamp 的拼接被传入了 sign，被 privateKey 进行了加密，然后得到的 sign。

```
sign is called, content: data=eyJwYXNzd29yZCI6ImQ4NTc4ZWRmODQ1OGNlMDZmYmM1YmI3NmE1OGM1Y2E0Iiwib3MiOiJhbmRyb2lkIiwibW9iaWxlIjoiMTU1MzYyNjM1MjIiLCJ2ZXJzaW9uIjoiMi4yLjMifQ==×tamp=1687276955,
privateKey: MIICdwIBADANBgkqhkiG9w0BAQEFAASCAmEwggJdAgEAAoGBAMNGABIfN+iron2hbwB7mLK1Dm05V1qLZBILTDj7dypr+GJzQ9fk0V7gIIchpFG7pDQEXMbb2nj8VkNAIIDBaw7UY1h9n+sCBT+Xzz6BB2UxLBBMQVwOwv55tJkZ2YBcHFQDGz51HjxAonKJdHwGpjIp7bwdx375gybn2ic4qNuFAgMBAAECgYEAmcha7eqgASCKCx5DaMHtc2+bOPFblfcIjB1Rnd6L7mCxb/cOisutB2bCtykLW0LHAiAdYI5r87Ply3iJIF0yjU35I8aieDVmeaQXXQfpisimXLOmz6p4VlBzAkz493oXPEH81cHqbwnFkiFE3VVtHbCNoZqXlFWthIdae2kpjlECQQDyCMl09eyDBNGuzg1r4tAQ4CeZe7aCkEFwK2at76Raqz9NKrynBiZHsKLU3JedRm2eZ7JimUhsuKbbkS/mcxBrAkEAzop7/PyddSXGDFDECyuXtuEKyzzUvdGiyNmOexhSwTmTZ7QdQqe5p382yCQcY8RXxZ6W9CLjuukfa9I6Tcz/zwJAQbjpG318D8fLOHBzbIxWe36iwia51JJfcpoWc7zTIFvIAKhOOfyNgIISdULBWM+7DHyUD/oXlI4/oPe3zhgIqQJAQd3gFJnrDQTy19KZ8oYAaA30h0PrBG3qX+shiRgErCJUY+oIus0KY+Qp8EGz3A0tgJRGx6you17E6nmspksN+QJBAKhaBGeHqs0+Z5wtFcunuqc6hV7WlhBYCe5YSxBNSiaohXDr6nQwjiOY22Q3m8aInp9KS+lDwW0o4C58VARKyo8=
sign ret value is Z5dDahgsBp06lh76v1fos8GSqX4HdcB+vJBJgw4qWN6wJ52T2LkX9e/WqtLrb+biSg10J23/f0KIQW57GFLIWEgLwQRWgKaacvJJVMtoeETAXhgMvYU9LWmjFzrtd1EuRLyHLVkWik7IbJM5mySllm+bG/nfcPvohXp6ATRqy7k=

```

然后 sign 的 python 还原，生成的结果与 frida 的 hook 结果相同，sign 完成，然后就是将已经实现的 python 代码进行拼接，并完成发包。

```
from Crypto.Hash import SHA
from Crypto.Signature import pkcs1_15
from Crypto.PublicKey import RSA
import base64
 
def sign(content, private_key):
    private_key_value = base64.b64decode(private_key)
    key = RSA.import_key(private_key_value)
    # 计算 SHA-1 哈希值
    hash_value = SHA.new(content.encode('utf-8'))
    # RSA 的钥长度为 1024 位
    k = 1024
    # 计算最大签名长度，根据 Java 的实现方式计算
    em_len = k // 4
    h_len = 20
    t_len = 3
    s_len = em_len - h_len - t_len - 1
    # 进行签名
    signature_value = pkcs1_15.new(key).sign(hash_value)[:s_len]
    # 返回 Base64 编码的结果
    return base64.b64encode(signature_value).decode()
content = "data=eyJwYXNzd29yZCI6ImQ4NTc4ZWRmODQ1OGNlMDZmYmM1YmI3NmE1OGM1Y2E0Iiwib3MiOiJhbmRyb2lkIiwibW9iaWxlIjoiMTU1MzYyNjM1MjIiLCJ2ZXJzaW9uIjoiMi4yLjMifQ==×tamp=1687276955"
private_key = 'MIICdwIBADANBgkqhkiG9w0BAQEFAASCAmEwggJdAgEAAoGBAMNGABIfN+iron2hbwB7mLK1Dm05V1qLZBILTDj7dypr+GJzQ9fk0V7gIIchpFG7pDQEXMbb2nj8VkNAIIDBaw7UY1h9n+sCBT+Xzz6BB2UxLBBMQVwOwv55tJkZ2YBcHFQDGz51HjxAonKJdHwGpjIp7bwdx375gybn2ic4qNuFAgMBAAECgYEAmcha7eqgASCKCx5DaMHtc2+bOPFblfcIjB1Rnd6L7mCxb/cOisutB2bCtykLW0LHAiAdYI5r87Ply3iJIF0yjU35I8aieDVmeaQXXQfpisimXLOmz6p4VlBzAkz493oXPEH81cHqbwnFkiFE3VVtHbCNoZqXlFWthIdae2kpjlECQQDyCMl09eyDBNGuzg1r4tAQ4CeZe7aCkEFwK2at76Raqz9NKrynBiZHsKLU3JedRm2eZ7JimUhsuKbbkS/mcxBrAkEAzop7/PyddSXGDFDECyuXtuEKyzzUvdGiyNmOexhSwTmTZ7QdQqe5p382yCQcY8RXxZ6W9CLjuukfa9I6Tcz/zwJAQbjpG318D8fLOHBzbIxWe36iwia51JJfcpoWc7zTIFvIAKhOOfyNgIISdULBWM+7DHyUD/oXlI4/oPe3zhgIqQJAQd3gFJnrDQTy19KZ8oYAaA30h0PrBG3qX+shiRgErCJUY+oIus0KY+Qp8EGz3A0tgJRGx6you17E6nmspksN+QJBAKhaBGeHqs0+Z5wtFcunuqc6hV7WlhBYCe5YSxBNSiaohXDr6nQwjiOY22Q3m8aInp9KS+lDwW0o4C58VARKyo8='
signature = sign(content, private_key)
print(signature)
#Z5dDahgsBp06lh76v1fos8GSqX4HdcB+vJBJgw4qWN6wJ52T2LkX9e/WqtLrb+biSg10J23/f0KIQW57GFLIWEgLwQRWgKaacvJJVMtoeETAXhgMvYU9LWmjFzrtd1EuRLyHLVkWik7IbJM5mySllm+bG/nfcPvohXp6ATRqy7k=

```

部分代码复现
======

![](https://bbs.kanxue.com/upload/attach/202307/973686_7JH8NYHCSZTWTJS.jpg)  
先实现上图的 login，实现手机号与密码的输入，传入 parems，形成相同的 login。  
代码如下:

```
import hashlib
def login(mobile, word):
    params = {
        "password": get_md5(word),
        "os": "android",
        "mobile": mobile,
        "version": "2.2.3",
    }
    return params
 
def get_md5(val):
    m = hashlib.md5()
    m.update(val.encode('utf-8'))
    result = m.hexdigest()
    return result
 
mobile = input("请输入16手机号:")
parem = input("请输入密码:")
params = login(mobile,parem)
print(params)
#运行结果：{'password': 'a7026e07535acfd00f0a8a4a97992291', 'os': 'android', 'mobile': '13685446446', 'version': '2.2.3'}

```

接着实现下图代码：  
![](https://bbs.kanxue.com/upload/attach/202307/973686_CCTAPHHMZD93X2R.jpg)  
代码如下：

```
def jiaparams(params):
    # 将参数转换为 JSON 字符串，并进行 Base64 编码
    params_json = json.dumps(params,separators=(',', ':'))
    print("json转成功：",params_json)
    data = base64.b64encode(params_json.encode('utf-8')).decode('utf-8')
    print("data成功:",data)
 
    # 对加密后的参数进行签名
    signstr = f"data={data}×tamp={times()}"
    print("时间戳：",times())
    print("拼接成功",signstr)
 
    signss = sign(signstr, pey)#sign加密的地方
    print("sign成功",signss)
 
    # 创建一个空字典用于存储加密后的参数
    result = {}
    result['data'] = data
    result['sign'] = signss
    result['timestamp'] = times()
    return result

```

最后代码拼接一起，得到下面代码：

```
import requests
  
import json
  
import time
  
import hashlib
  
from Crypto.Hash import SHA
  
from Crypto.Signature import pkcs1_15
  
from Crypto.PublicKey import RSA
  
import base64
  
  
  
pey = 'MIICdwIBADANBgkqhkiG9w0BAQEFAASCAmEwggJdAgEAAoGBAMNGABIfN+iron2hbwB7mLK1Dm05V1qLZBILTDj7dypr+GJzQ9fk0V7gIIchpFG7pDQEXMbb2nj8VkNAIIDBaw7UY1h9n+sCBT+Xzz6BB2UxLBBMQVwOwv55tJkZ2YBcHFQDGz51HjxAonKJdHwGpjIp7bwdx375gybn2ic4qNuFAgMBAAECgYEAmcha7eqgASCKCx5DaMHtc2+bOPFblfcIjB1Rnd6L7mCxb/cOisutB2bCtykLW0LHAiAdYI5r87Ply3iJIF0yjU35I8aieDVmeaQXXQfpisimXLOmz6p4VlBzAkz493oXPEH81cHqbwnFkiFE3VVtHbCNoZqXlFWthIdae2kpjlECQQDyCMl09eyDBNGuzg1r4tAQ4CeZe7aCkEFwK2at76Raqz9NKrynBiZHsKLU3JedRm2eZ7JimUhsuKbbkS/mcxBrAkEAzop7/PyddSXGDFDECyuXtuEKyzzUvdGiyNmOexhSwTmTZ7QdQqe5p382yCQcY8RXxZ6W9CLjuukfa9I6Tcz/zwJAQbjpG318D8fLOHBzbIxWe36iwia51JJfcpoWc7zTIFvIAKhOOfyNgIISdULBWM+7DHyUD/oXlI4/oPe3zhgIqQJAQd3gFJnrDQTy19KZ8oYAaA30h0PrBG3qX+shiRgErCJUY+oIus0KY+Qp8EGz3A0tgJRGx6you17E6nmspksN+QJBAKhaBGeHqs0+Z5wtFcunuqc6hV7WlhBYCe5YSxBNSiaohXDr6nQwjiOY22Q3m8aInp9KS+lDwW0o4C58VARKyo8='
  
  
  
url = 'https://api.langshiyu.com/user/v2/account/login'
  
  
  
def md5(pwd_str):
  
    md5_obj = hashlib.md5()
  
    md5_obj.update(pwd_str.encode('utf-8'))
  
    return md5_obj.hexdigest()
  
  
  
def times():
  
    return str(int(time.time()))
  
  
  
def sign(content, private_key):
  
    private_key_value = base64.b64decode(private_key)
  
    key = RSA.import_key(private_key_value)
  
  
  
    # 计算 SHA-1 哈希值
  
    hash_value = SHA.new(content.encode('utf-8'))
  
  
  
    # RSA 的密钥长度为 1024 位
  
    k = 1024
  
  
  
    # 计算最大签名长度，根据 Java 的实现方式计算
  
    em_len = k // 4
  
    h_len = 20
  
    t_len = 3
  
    s_len = em_len - h_len - t_len - 1
  
  
  
    # 进行签名
  
    signature_value = pkcs1_15.new(key).sign(hash_value)[:s_len]
  
  
  
    # 返回 Base64 编码的结果
  
    return base64.b64encode(signature_value).decode()
  
  
  
def login(mobile, param):
  
    params = {
  
        "password": md5(param),
  
        "os": "android",
  
        "mobile": mobile,
  
        "version": "2.2.3",
  
    }
  
    return params
 
def jiaparams(params):
  
    # 将参数转换为 JSON 字符串，并进行 Base64 编码
  
    params_json = json.dumps(params,separators=(',', ':'))
  
    print("json转成功：",params_json)
  
    data = base64.b64encode(params_json.encode('utf-8')).decode('utf-8')
  
    print("data成功:",data)
  
    # 对加密后的参数进行签名
  
    signstr = f"data={data}×tamp={times()}"
  
    print("时间戳：",times())
  
    print("拼接成功",signstr)
 
    signss = sign(signstr, pey)#sign加密的地方
  
    print("sign成功",signss)
 
    # 创建一个空字典用于存储加密后的参数
  
    result = {}
  
    result['data'] = data
  
    result['sign'] = signss
  
    result['timestamp'] = times()
  
    return result
  
mobile = input("请输入16位手机号:")
  
parem = input("请输入密码:")
  
params = login(mobile,parem)
  
paramss = jiaparams(params)
  
print("发包内容：",paramss)
  
fb= requests.post(url, paramss)
  
print("结果：",fb.text)bjsdm
  
#结果：{"code":0,"message":"该账户不存在","reqdata":{"data":0},"reqtime":"0.015238"}

```

最后总结：
=====

有部分简单的代码实现，以及开头通过 objection 的 SSL 屏蔽后 charles 进行抓包进行找的关键点，都没有在文章里面写，不过这些都是不必要的，所以文章是有一小部分省略。  
整体对小白非常友好的软件，难度不算很高，只要花时间就可以掌握关键点，进行复现。  
app 放在有道云文章里面，想尝试一下的小白可以进去拿一下。  
算法还原过程有道云文章链接：[https://note.youdao.com/s/Yf9jGzqk](https://note.youdao.com/s/Yf9jGzqk)

[二进制系列之 Pwn 篇](https://www.kanxue.com/book-section_list-148.htm)

[#逆向分析](forum-161-1-118.htm)