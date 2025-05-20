> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286892.htm)

> [原创]BLE 安全初探: 流量分析和安全加固

[原创]BLE 安全初探: 流量分析和安全加固

发表于: 12 小时前 102

[举报](javascript:void(0);)

### [原创]BLE 安全初探: 流量分析和安全加固

 [![](https://bbs.kanxue.com/view/img/avatar.png)](user-home-1014504.htm) [CLan_nad](user-home-1014504.htm) ![](https://bbs.kanxue.com/view/img/rank/4.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 12 小时前  102

前言
==

前段时间出于个人兴趣和需求，买了 esp32 开发板做了个蓝牙控制舵机，安装在宿舍灯旁，实现了手机远程关宿舍灯。

最近出于对 ble 安全的学习目的，买了款 nrf52840-dongle，对开发板的 ble 服务做嗅探，可以由此初步学习对于 ble 的流量分析和安全加固。

基础介绍
====

ble 基础
------

蓝牙是一种短距的无线通讯技术，BLE（Bluetooh Low Energy）可视为蓝牙技术的一个新模块，并区别于传统模块，最大的特点就是成本和功耗降低，应用于实时性要求比较高的场景。

BLE 协议栈如下

![](https://bbs.kanxue.com/upload/attach/202505/1014504_NU4XZBZNPXGTCXE.jpg)

后续的流量分析中会涉及到的协议层如下

1、LL 层（Link Layer 链路层）

ble 是面向连接的协议，类 tcp。连接前需要进行广播，不断在广播通道发送广播报文。包括广播报文和建立连接后的报文都是以链路层为基础的。

1.  2、L2CAP 层（Logic link control and adaptation protocol）  
    L2CAP 对 LL 进行了一次简单封装，LL 只关心传输的数据本身，L2CAP 就要区分是加密通道还是普通通道，同时还要对连接间隔进行管理。
    
2.  3、SMP（Secure manager protocol）  
    SMP 用来管理 BLE 配对过程的，保证连接的安全性，过程包括密钥生成与分发。配对分为传统配对（legacy pairing）和安全配对（LE Secure Connections）
    
3.  4、ATT（Attribute protocol）
    
4.  ATT 层用来定义用户命令及命令操作的数据，比如读取某个数据或者写某个数据。BLE 引入了 attribute 概念，用来描述一条一条的数据。Attribute 除了定义数据，同时定义该数据可以使用的 ATT 命令，因此这一层被称为 ATT 层。
    

5、GATT（Generic attribute profile ）

GATT 用来规范 attribute 中的数据内容，并运用 group（分组）的概念对 attribute 进行分类管理。GATT 中存在服务端和客户端的定义，在 GATT 定义了服务（service），每个服务可以包含零个或多个特征（characteristic），不同的特征之间用唯一的 UUID 区分，这些特征又可以包括零个或多个：

描述符（descriptor），顾名思义，可有可无。

值（value），就是操作行为的实际数据。

属性（properties），指定了读或写。

ESP32 的 BLE 开发
--------------

硬件的工作不做赘述。以下是刷入 esp32 的代码，是不进行安全加固的版本。

（ps：本人是嵌入式开发的小白，esp32 开发的门槛不高，使用到了 arduino ide，它提供了简化开发的库函数和框架）。  

```
#include  #include  #include  #include  // 使用 ESP32 专用的舵机库
  
#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"
#define SERVO_PIN 13  // 舵机信号线接 GPIO13
  
Servo myservo;  // 使用 ESP32Servo 库创建舵机对象
bool deviceConnected = false;
BLEServer* pServer = nullptr;
BLECharacteristic* pChar = nullptr;
 
//接收到特征值写入的回调
class MyCallbacks : public BLECharacteristicCallbacks {
    void onWrite(BLECharacteristic *pCharacteristic) {
        std::string value = pCharacteristic->getValue();
        if (value == "OFF") {           
            myservo.write(100);          
            delay(1000);                
            myservo.write(50);
            delay(1000);
            myservo.write(10);
            delay(1000);
            myservo.write(50);
        } 
    }
};
  
  
//BLE服务创建的回调
class MyServerCallbacks : public BLEServerCallbacks {
    void onConnect(BLEServer* pServer) {
        deviceConnected = true;
    }
  
    void onDisconnect(BLEServer* pServer) {
        deviceConnected = false;
    }
};
  
void setup() {
    Serial.begin(115200);  // 初始化串口通信
    myservo.attach(SERVO_PIN);  // 将舵机绑定到 GPIO13
    myservo.write(50);
    BLEDevice::init("SmartLight_BLE"); 
    pServer = BLEDevice::createServer();
    pServer->setCallbacks(new MyServerCallbacks());
    BLEService *pService = pServer->createService(SERVICE_UUID);
    pChar = pService->createCharacteristic(
        CHARACTERISTIC_UUID,
        BLECharacteristic::PROPERTY_WRITE
    );
    pChar->setCallbacks(new MyCallbacks());
    pService->start();
    pServer->getAdvertising()->start();
    Serial.println("BLE 广播开始，等待连接...");
}
  
void loop() {
    if (!deviceConnected) {
        // 如果设备断开连接，重新启动广播
        pServer->getAdvertising()->start();
    }
    delay(2000);
     
} 
```

以上代码烧入 esp32，它就作为一个 gatt 的服务端运行，开启了 ble 服务，不断广播等待连接，并创建了特征，定义了收到特定的特征值 "OFF"，即进行关灯操作。此时手机作为客户端，与服务器连接，通过 nrfconnect 这个 app 可以发送自己编辑的特征值。

除了服务端 - 客户端的叫法，也可以叫做从机 - 主机，外设（ Peripheral ）- 中心设备（ Central）。

nrf52840 的使用
------------

nrf52840 是一款支持 ble 工作的芯片，可以用作蓝牙扫描器或嗅探器，配合 wireshark 捕获和分析周围设备的 BLE 广播包和连接包。价格 40 左右。

配置过程参考 [Wireshark 配合 nRF Sniffer 使用技巧 - unrulife - 博客园 (cnblogs.com)](elink@b46K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2U0L8X3u0D9L8$3N6K6i4K6u0W2j5$3!0E0i4K6u0r3N6h3&6J5N6h3I4A6k6X3g2Q4x3V1k6H3i4K6u0r3x3e0l9^5x3e0R3J5y4e0W2Q4x3X3g2Z5N6r3#2D9)

流量分析
====

开始捕获后，在连接前有大量的广播包，info 显示为 ADV_IND 等信息。  

![](https://bbs.kanxue.com/upload/attach/202505/1014504_PSFZUMVFDJ89AK6.webp)

手机向设备发起配对请求后，可以看到首先有个由手机发起的连接请求，info 为 CONNECT_IND。随后没有发现有效数据，同时关注到存在 "Encrypted packet decrypted incorrectly (bad MIC)" 这样的字眼，说明流量被加密了。

![](https://bbs.kanxue.com/upload/attach/202505/1014504_VS4ZJSJHZ5F8ZZJ.webp)

用 btsmp 过滤出 SMP 包，说明配对的过程是会通过 SMP 协议进行密钥的管理的。

客户端发送一个 Pairing Request，服务端返回一个 Pairing Response，还有一些随机数的交换和确认。最后产生一个 Long Term Key（LTK），用于加密配对后的通信。

![](https://bbs.kanxue.com/upload/attach/202505/1014504_H5R6GBK8HDEVHBN.webp)

Pairing Response 的关键字段如下：

1、IO Capability

是根据输入输出能力协商配对方式，No Input, No Output (0x03)  
声明设备无输入输出界面，因此使用 "Just Works" 或 静态密码 的配对方式。

### 2、OOB Data Flag （带外数据标志）

声明配对过程中是否使用 带外认证数据（如 NFC、二维码等非 BLE 通道传输的密钥）  

为 0 说明不启用  

##### 3、AuthReq（认证请求标志）

0x09 的二进制解析

<table width="NaN"><thead><tr><th>参数名称</th><th>值</th><th>描述</th></tr></thead><tbody><tr><td>Secure Connection Flag</td><td>1</td><td>启用安全连接（AES-CCM 加密，防窃听）</td></tr><tr><td>MITM（中间人防护）</td><td>0</td><td>未启用中间人攻击防护</td></tr><tr><td>Bonding（绑定）</td><td>1</td><td>启用绑定，存储 LTK（长 - term key）以便后续连接使用</td></tr></tbody></table>  

这里是启用了 Secure Connection 模式的，即上文提到的更先进的安全配对，如果是传统配对（legacy pairing），是一个存在漏洞，已经被破解了的协议，参考 [GitHub - mikeryan/crackle: Crack and decrypt BLE encryption](elink@0feK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6E0K9h3E0W2M7Y4W2S2L8W2)9J5c8X3y4J5j5h3y4C8L8r3f1`.)。

![](https://bbs.kanxue.com/upload/attach/202505/1014504_R5E277B4R3EMCCB.webp)  

流量嗅探到这一步就卡住了。我一开始 esp32 中的代码并未进行安全加固，但这里抓包就看出已经启用了 SMP 来加密通信流量。一般来说，如果服务端代码中不显式调用  

```
BLEDevice::setEncryptionLevel(ESP_BLE_SEC_ENCRYPT_MITM);
BLEDevice::setSecurityCallbacks(new MySecurity());
```

等代码，连接时应该是不会启用 SMP 配对的。这里一翻折腾后，尝试想着怎么获取 LTK 或者解密流量都无果。最后发现，是我的安卓手机的原因，因为手机的安全机制要求，需要先强制进行配对，然后后续再建立连接进行通信。

而 nrfconnect 这个 app 有一种方法可以绕开配对直接进行连接：先配对完成后，在更多选项中删去绑定信息，即断开了配对，删除了加密的缓存。此时连接的功能可以继续使用，点击连接即可无加密通信了。

此时发起连接，并没有出现 SMP 协议包，加密也未启用。

![](https://bbs.kanxue.com/upload/attach/202505/1014504_VDNEP9K39ZKMEDC.webp)

发送关灯指令后，输入 btatt，过滤 ATT 协议，找到了关灯指令发送的 ATT 数据包，即嗅探操作已经完成了。下图标注了 BLE 服务的 uuid，特征的 uuid，还有特征值 value 未 4f4646，即 "OFF"。

![](https://bbs.kanxue.com/upload/attach/202505/1014504_MEDKJD647YEP6TS.webp)

如果是模拟攻击行为。嗅探到 ATT 数据之后就可以进行重放攻击，这里是比较简单的情况，使用手机上的 nrf connect 就可以完成重放了。而复杂一点的情况，比如涉及到开发 exp 脚本，或者对实际程序逆向后发现特征值构造比较复杂，可能需要的工具是 kali 中的 gatttool。

安全加固  

=======

分析中知道了，如果设备中代码本身不启用安全配置的话，ble 客户端是可以无认证直接连接的，还能进行流量嗅探，进行重放甚至中间人攻击。（客户端 ble 往往是手机 app，有很多实际案例中，连接 ble 设备前是不会经过安卓手机本身要求的安全配对的，而是直接发起连接）  

代码层面进行安全加固的一个例子如下

```
  // ==================== BLE安全认证回调 ====================
class MySecurityCallbacks : public BLESecurityCallbacks {
    uint32_t onPassKeyRequest() {
        Serial.println("手机请求配对码");
        return STATIC_PIN; // 返回静态配对码
    }
 
    void onAuthenticationComplete(esp_ble_auth_cmpl_t cmpl) {
        if (cmpl.success) {
            Serial.println("配对成功，通信已加密");
        } else {
            Serial.println("配对失败");
        }
    }
};
 
 void setup(){
    ...
    BLEDevice::setEncryptionLevel(ESP_BLE_SEC_ENCRYPT_MITM); // 启用加密和MITM保护
    BLEDevice::setSecurityCallbacks(new MySecurityCallbacks());  // 注册安全回调
     
    BLESecurity *pSecurity = new BLESecurity();
    pSecurity->setAuthenticationMode(ESP_LE_AUTH_REQ_MITM_BOND); // 要求配对和绑定
    pSecurity->setStaticPIN(STATIC_PIN);             //  设置静态配对码                      
      
    ...
     
 }
```

这是静态认证的方式，只允许通过认证的设备连接。  

BLEDevice::setEncryptionLevel() 函数有多个等级

<table width="NaN"><thead><tr><th><strong>加密等级</strong></th><th><strong>宏定义</strong></th><th><strong>说明</strong></th></tr></thead><tbody><tr><td><strong>无加密</strong></td><td><code>ESP_BLE_SEC_NONE</code></td><td>不加密，明文通信（不安全）</td></tr><tr><td><strong>仅加密（无认证）</strong></td><td><code>ESP_BLE_SEC_ENCRYPT</code></td><td>仅加密数据，但不验证设备身份（防窃听，但不防 MITM）</td></tr><tr><td><strong>加密 + MITM 保护</strong></td><td><code>ESP_BLE_SEC_ENCRYPT_MITM</code></td><td>加密 + 防中间人攻击（需配对码或 OOB 认证）</td></tr></tbody></table>

除了静态密码认证的方式，还有交互认证等方式，这种情况会需要数据的输入输出外设。  

尾言
==

可以看出，ble 设备的安全依赖协议安全和开发者的配置，非敏感 ble 设备的开发者的安全意识或许不太强，嗅探重放攻击的攻击面可能会有一定价值，同时可能会需要 app 逆向的工作，解析实际设备的特征值含义。

  

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#基础理论](forum-128-1-165.htm) [#家用设备](forum-128-1-173.htm) [#安全研究](forum-128-1-167.htm)