> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283299.htm)

> [原创] 淺談 Cocos2djs 逆向

前言
--

簡單聊一下 cocos2djs 手遊的逆向，有任何相關想法歡迎和我討論 ^^

一些概念
----

列出一些個人認為比較有用的概念：

*   Cocos 遊戲的兩大開發工具分別是`CocosCreator`和`CocosStudio`，區別是前者是 cocos2djs 專用的開發工具，後者則是 cocos2d-lua、cocos2d-cpp 那些。

![](https://bbs.kanxue.com/upload/attach/202409/946537_K3NQFC6REHUET33.webp)

*   使用`Cocos Creator 2`開發的手遊，生成的關鍵 so 默認名稱是`libcocos2djs.so`
*   使用`Cocos Creator 3`開發的手遊，生成的關鍵 so 默認名稱是`libcocos.so` ( 入口函數非`applicationDidFinishLaunching` )
*   Cocos Creator 在構建時可以選擇是否對`.js`腳本進行加密 & 壓縮，而加密算法固定是`xxtea`，還可以選擇是否使用 Zip 壓縮

![](https://bbs.kanxue.com/upload/attach/202409/946537_MRGS5M4K93GX3WM.webp)

*   `libcocos2djs.so`裡的`AppDelegate::applicationDidFinishLaunching`是入口函數，可以從這裡開始進行分析
*   Cocos2djs 是 Cocos2d-x 的一個分支，因此 [https://github.com/cocos2d/cocos2d-x 源碼同樣適用於 Cocos2djs](https://github.com/cocos2d/cocos2d-x%E6%BA%90%E7%A2%BC%E5%90%8C%E6%A8%A3%E9%81%A9%E7%94%A8%E6%96%BCCocos2djs)

自己寫一個 Demo
----------

自己寫一個 Demo 來分析的好處是能夠快速地判斷某個錯誤是由於被檢測到？還是本來就會如此？

### 版本信息

嘗試過 2.4.2、2.4.6 兩個版本，都構建失敗，最終成功的版本信息如下：

*   編輯器版本：`Creator 2.4.13` (2 系列裡的最高版本，低版本在 AS 編譯時會報一堆錯誤)
*   ndk 版本：`23.1.7779620`
*   `project/build.gradle`：`classpath 'com.android.tools.build:gradle:8.0.2'`
*   `project/gradle/gradle-wrapper.properties`：`distributionUrl=https\://services.gradle.org/distributions/gradle-8.0.2-all.zip`

### Cocos Creator 基礎用法

由於本人不懂 cocos 遊戲開發，只好直接用官方的 Hello World 模板。

![](https://bbs.kanxue.com/upload/attach/202409/946537_NDNDJYH7DJM4689.webp)

首先要設置 SDK 和 NDK 路徑

![](https://bbs.kanxue.com/upload/attach/202409/946537_BFWBHHPQ4S4HFDY.webp)

然後構建的參數設置如下，主要需要設置以下兩點：

*   加密腳本：全都勾上，密鑰用默認的
*   Source Map：保留符號，這樣 IDA 在打開時才能看到函數名

![](https://bbs.kanxue.com/upload/attach/202409/946537_QRFVCM7ZTJ7K33K.webp)

我使用 Cocos Creator 能順利構建，但無法編譯，只好改用 Android Studio 來編譯。

使用 Android Studio 打開`build\jsb-link\frameworks\runtime-src\proj.android-studio`，然後就可以按正常 AS 流程進行編譯

Demo 如下所示，在中心輸出了`Hello, World!`。

![](https://bbs.kanxue.com/upload/attach/202409/946537_88ZTD32TDCYBZMU.webp)

jsc 腳本解密
--------

上述 Demo 構建中有一個選項是【加密腳本】，它會將 js 腳本通過 xxtea 算法加密成`.jsc`。

而遊戲的一些功能就會通過 js 腳本來實現，因此 cocos2djs 逆向首要事件就是將`.jsc`解密，通常`.jsc`會存放在 apk 內的`assets`目錄下

![](https://bbs.kanxue.com/upload/attach/202409/946537_TD96UHCGK98YEEH.webp)

### 獲取解密 key

方法一：從`applicationDidFinishLaunching`入手

![](https://bbs.kanxue.com/upload/attach/202409/946537_6WP6MHN6MQB34B2.webp)

方法二：HOOK

1.  hook `set_xxtea_key`

```
// soName: libcocos2djs.so
function hook_jsb_set_xxtea_key(soName) {
    let set_xxtea_key = Module.findExportByName(soName, "_Z17jsb_set_xxtea_keyRKNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEE");
    Interceptor.attach(set_xxtea_key,{
        onEnter(args){
            console.log("xxtea key: ", args[0].readCString())
        },
        onLeave(retval){

        }
    })
}
```

1.  hook `xxtea_decrypt`

```
function hook_xxtea_decrypt(soName) {
    let set_xxtea_key = Module.findExportByName(soName, "xxtea_decrypt");
    Interceptor.attach(set_xxtea_key,{
        onEnter(args){
            console.log("xxtea key: ", args[2].readCString())
        },
        onLeave(retval){

        }
    })
}
```

### python 加解密腳本

一次性解密`output_dir`目錄下所有`.jsc`，並在`input_dir`生成與`output_dir`同樣的目錄結構。

```
# pip install xxtea-py
# pip install jsbeautifier
 
import xxtea
import gzip
import jsbeautifier
import os
 
KEY = "abdbe980-786e-45"
 
input_dir = r"cocos2djs_demo\assets" # abs path
 
output_dir = r"cocos2djs_demo\output" # abs path
 
def jscDecrypt(data: bytes, needJsBeautifier = True):
    dec = xxtea.decrypt(data, KEY)
    jscode = gzip.decompress(dec).decode()
 
    if needJsBeautifier:
        return jsbeautifier.beautify(jscode)
    else:
        return jscode
 
def jscEncrypt(data):
    compress_data = gzip.compress(data.encode())
    enc = xxtea.encrypt(compress_data, KEY)
    return enc
 
def decryptAll():
    for root, dirs, files in os.walk(input_dir):
         
        # 創建與input_dir一致的結構
        for dir in dirs:
            dir_path = os.path.join(root, dir)
            target_dir = output_dir + dir_path.replace(input_dir, "")
            if not os.path.exists(target_dir):
                os.mkdir(target_dir)
 
        for file in files:
            file_path = os.path.join(root, file)
        
            if not file.endswith(".jsc"):
                continue
             
            with open(file_path, mode = "rb") as f:
                enc_jsc = f.read()
             
            dec_jscode = jscDecrypt(enc_jsc)
             
            output_file_path = output_dir + file_path.replace(input_dir, "").replace(".jsc", "") + ".js"
 
            print(output_file_path)
            with open(output_file_path, mode = "w", encoding = "utf-8") as f:
                f.write(dec_jscode)
 
def decryptOne(path):
    with open(path, mode = "rb") as f:
        enc_jsc = f.read()
     
    dec_jscode = jscDecrypt(enc_jsc, False)
 
    output_path = path.split(".jsc")[0] + ".js"
 
    with open(output_path, mode = "w", encoding = "utf-8") as f:
        f.write(dec_jscode)
 
def encryptOne(path):
    with open(path, mode = "r", encoding = "utf-8") as f:
        jscode = f.read()
 
    enc_data = jscEncrypt(jscode)
     
    output_path = path.split(".js")[0] + ".jsc"
 
    with open(output_path, mode = "wb") as f:
        f.write(enc_data)
 
if __name__ == "__main__":
    decryptAll()
```

`jsc`文件的 2 種讀取方式
----------------

為實現對遊戲正常功能的干涉，顯然需要修改遊戲執行的 js 腳本。而替換`.jsc`文件是其中一種思路，前提是要找到讀取`.jsc`文件的地方。

### 方式一：從 apk 裡讀取

我自己編譯的 Demo 就是以這種方式讀取`/data/app/XXX/base.apk`裡`assets`目錄內的`.jsc`文件。

cocos 引擎默認使用 xxtea 算法來對`.jsc`等腳本進行加密，因此讀取`.jsc`的操作定然在`xxtea_decrypt`之前。

跟 [cocos2d-x 源碼](https://github.com/cocos2d/cocos2d-x/blob/76903dee64046c7bfdba50790be283484b4be271/cocos/scripting/lua-bindings/manual/CCLuaStack.cpp#L782)，找使用`xxtea_decrypt`的地方，可以定位到`LuaStack::luaLoadChunksFromZIP`

![](https://bbs.kanxue.com/upload/attach/202409/946537_D56DGTMMQHPVNMT.webp)

向上跟會發現它的 bytes 數據是由`getDataFromFile`函數獲取

![](https://bbs.kanxue.com/upload/attach/202409/946537_5HUNC7BU5232XJY.webp)

繼續跟`getDataFromFile`的邏輯，它會調用`getContents`，而`getContents`裡是調用`fopen`來打開，但奇怪的是 hook `fopen`卻沒有發現它有打開任何`.jsc`文件

![](https://bbs.kanxue.com/upload/attach/202409/946537_JAZ4QFQEZ79VFW6.webp)

![](https://bbs.kanxue.com/upload/attach/202409/946537_7KUFT82MNF3P4YY.webp)

後來發現調用的並非`FileUtils::getContents`，而是`FileUtilsAndroid::getContents`。

它其中一個分支是調用`libandroid.so`的`AAsset_read`來讀取`.jsc`數據，調用`AAssetManager_open`來打開`.jsc`文件。

![](https://bbs.kanxue.com/upload/attach/202409/946537_FGE9ZD7GHYW98EK.webp)

繼續對`AAssetManager_open`進行深入分析 ( [在線源碼](http://xrefandroid.com/android-8.1.0_r81/xref/frameworks/base/native/android/asset_manager.cpp?r=&mo=2286&fi=87#87) )，目的是找到能夠 IO 重定向的點：

`AAssetManager_open`裡調用了`AssetManager::open`函數

```
// frameworks/base/native/android/asset_manager.cpp
AAsset* AAssetManager_open(AAssetManager* amgr, const char* filename, int mode)
{
    Asset::AccessMode amMode;
    switch (mode) {
    case AASSET_MODE_UNKNOWN:
        amMode = Asset::ACCESS_UNKNOWN;
        break;
    case AASSET_MODE_RANDOM:
        amMode = Asset::ACCESS_RANDOM;
        break;
    case AASSET_MODE_STREAMING:
        amMode = Asset::ACCESS_STREAMING;
        break;
    case AASSET_MODE_BUFFER:
        amMode = Asset::ACCESS_BUFFER;
        break;
    default:
        return NULL;
    }
 
    AssetManager* mgr = static_cast(amgr);
    // here
    Asset* asset = mgr->open(filename, amMode);
    if (asset == NULL) {
        return NULL;
    }
 
    return new AAsset(asset);
} 
```

`AssetManager::open`調用`openNonAssetInPathLocked`

```
// frameworks/base/libs/androidfw/AssetManager.cpp
Asset* AssetManager::open(const char* fileName, AccessMode mode)
{
    AutoMutex _l(mLock);
    LOG_FATAL_IF(mAssetPaths.size() == 0, "No assets added to AssetManager");
    String8 assetName(kAssetsRoot);
    assetName.appendPath(fileName);
 
    size_t i = mAssetPaths.size();
    while (i > 0) {
        i--;
        ALOGV("Looking for asset '%s' in '%s'\n",
                assetName.string(), mAssetPaths.itemAt(i).path.string());
        // here
        Asset* pAsset = openNonAssetInPathLocked(assetName.string(), mode, mAssetPaths.itemAt(i));
        if (pAsset != NULL) {
            return pAsset != kExcludedAsset ? pAsset : NULL;
        }
    }
 
    return NULL;
}
```

`AssetManager::openNonAssetInPathLocked`先判斷`assets`是位於`.gz`還是`.zip`內，而`.apk`與`.zip`基本等價，因此理應會走 else 分支。

```
奇怪的是當我使用frida hook驗證時，能順利hook到`openAssetFromZipLocked`，卻hook不到`getZipFileLocked`，顯然是不合理的。
```

```
// frameworks/base/libs/androidfw/AssetManager.cpp
Asset* AssetManager::openNonAssetInPathLocked(const char* fileName, AccessMode mode,
    const asset_path& ap)
{
    Asset* pAsset = NULL;
 
    if (ap.type == kFileTypeDirectory) {
        String8 path(ap.path);
        path.appendPath(fileName);
 
        pAsset = openAssetFromFileLocked(path, mode);
 
        if (pAsset == NULL) {
            /* try again, this time with ".gz" */
            path.append(".gz");
            pAsset = openAssetFromFileLocked(path, mode);
        }
 
        if (pAsset != NULL) {
            //printf("FOUND NA '%s' on disk\n", fileName);
            pAsset->setAssetSource(path);
        }
 
    // run this branch
    } else {
        String8 path(fileName);
                // here
        ZipFileRO* pZip = getZipFileLocked(ap);
        if (pZip != NULL) {
 
            ZipEntryRO entry = pZip->findEntryByName(path.string());
            if (entry != NULL) {
                 
                pAsset = openAssetFromZipLocked(pZip, entry, mode, path);
                pZip->releaseEntry(entry);
            }
        }
 
        if (pAsset != NULL) {
            pAsset->setAssetSource(
                    createZipSourceNameLocked(ZipSet::getPathName(ap.path.string()), String8(""),
                                                String8(fileName)));
        }
    }
 
    return pAsset;
}
```

嘗試繼續跟剛剛 hook 失敗的`AssetManager::getZipFileLocked`，它調用的是`AssetManager::ZipSet::getZip`。

```
同樣用frida hook `getZip`，這次成功了，猜測是一些優化移除了`getZipFileLocked`而導致hook 失敗。
```

```
// frameworks/base/libs/androidfw/AssetManager.cpp
ZipFileRO* AssetManager::getZipFileLocked(const asset_path& ap)
{
    ALOGV("getZipFileLocked() in %p\n", this);
 
    return mZipSet.getZip(ap.path);
}
```

`ZipSet::getZip`會調用`SharedZip::getZip`，後者直接返回`mZipFile`。

```
// frameworks/base/libs/androidfw/AssetManager.cpp
ZipFileRO* AssetManager::ZipSet::getZip(const String8& path)
{
    int idx = getIndex(path);
    sp zip = mZipFile[idx];
    if (zip == NULL) {
        zip = SharedZip::get(path);
        mZipFile.editItemAt(idx) = zip;
    }
    return zip->getZip();
}
 
ZipFileRO* AssetManager::SharedZip::getZip()
{
    return mZipFile;
} 
```

尋找`mZipFile`賦值的地方，最終會找到是由`ZipFileRO::open(mPath.string())`賦值。

```
// frameworks/base/libs/androidfw/AssetManager.cpp
AssetManager::SharedZip::SharedZip(const String8& path, time_t modWhen)
    : mPath(path), mZipFile(NULL), mModWhen(modWhen),
      mResourceTableAsset(NULL), mResourceTable(NULL)
{
    if (kIsDebug) {
        ALOGI("Creating SharedZip %p %s\n", this, (const char*)mPath);
    }
    ALOGV("+++ opening zip '%s'\n", mPath.string());
    // here
    mZipFile = ZipFileRO::open(mPath.string());
    if (mZipFile == NULL) {
        ALOGD("failed to open Zip archive '%s'\n", mPath.string());
    }
}
```

```
從`frameworks/base/libs/androidfw/Android.bp`可知上述代碼的lib文件是`libandroidfw.so`，位於`/system/lib64/`下。將其pull到本地然後用IDA打開，就能根據IDA所示的函數導出名稱/地址對這些函數進行hook。
```

### 方式二：從應用的數據目錄裡讀取

無論是方式一還是方式二，`.jsc`數據都是通過`getDataFromFile`獲取。而`getDataFromFile`裡調用了`getContents`。

```
getDataFromFile -> getContents
```

在方式一中，我一開始看的是`FileUtils::getContents`，但其實是`FileUtilsAndroid::getContents`才對。

只有當`fullPath[0] == '/'`時才會調用`FileUtils::getContents`，而`FileUtils::getContents`會調用`fopen`來打開`.jsc`

```
// https://github.com/cocos2d/cocos2d-x/blob/76903dee64046c7bfdba50790be283484b4be271/cocos/platform/android/CCFileUtils-android.cpp
FileUtils::Status FileUtilsAndroid::getContents(const std::string& filename, ResizableBuffer* buffer) const
{
    static const std::string apkprefix("assets/");
    if (filename.empty())
        return FileUtils::Status::NotExists;
 
    string fullPath = fullPathForFilename(filename);
 
    if (fullPath[0] == '/')
            // here
        return FileUtils::getContents(fullPath, buffer);
         
    // 方式一會走這裡....
}
```

替換思路
----

正常來說有以下幾種替換腳本的思路：

1.  找到讀取`.jsc`文件的地方進行 IO 重定向。
    
2.  直接進行字節替換，即替換`xxtea_decypt`解密前的`.jsc`字節數據，或者替換`xxtea_decypt`解密後的明文`.js`腳本。
    
    這裡的替換是指開闢一片新內存，將新的數據放到這片內存，然後替換指針的指向。
    
3.  直接替換 apk 裡的`.jsc`，然後重打包 apk。
    
4.  替換 js 明文，不是像`2`那樣開闢一片新內存，而是直接修改原本內存的明文 js 數據。
    

經測試後發現只有`1`、`3`、`4`是可行的，`2`會導致 APP 卡死 (原因不明？？？)。

### 思路一實現

從上述可知第一種`.jsc`讀取方式會先調用`ZipFileRO::open(mPath.string())`來打開 apk，之後再通過`AAssetManager_open`來獲取`.jsc`。

hook `ZipFileRO::open`看看傳入的參數是什麼。

```
function hook_ZipFile_open(flag) {
    let ZipFile_open = Module.getExportByName("libandroidfw.so", "_ZN7android9ZipFileRO4openEPKc"); 
    console.log("ZipFile_open: ", ZipFile_open)
    return Interceptor.attach(ZipFile_open,
        {
            onEnter: function (args) {
                console.log("arg0: ", args[0].readCString());
            },
            onLeave: function (retval) {

            }
        }
    );
}
```

可以看到其中一條是當前 APK 的路徑，顯然`assets`也是從這裡取的，因此這裡是一個可以嘗試重定向點，先需構造一個`fake.apk` push 到`/data/app/XXX/`下，然後 hook IO 重定向到`fake.apk`實現替換。

![](https://bbs.kanxue.com/upload/attach/202409/946537_Y5RDEYHURGYHDGF.webp)

對我自己編譯的 Demo 而言，無論是以 apktool 解包 & 重打包的方式，還是直接解壓縮 & 重壓縮 & 手動命名的方式來構建`fake.apk`都是可行的，**但要記得賦予`fake.apk`最低`644`的權限。**

以下是我使用上述方法在我的 Demo 中實踐的效果，成功修改中心的字符串。

![](https://bbs.kanxue.com/upload/attach/202409/946537_MMC9945TCA2N3MM.webp)

但感覺這種方式的實用性較低 (什至不如直接重打包…)

### 思路二嘗試 (失敗)

連這樣僅替換指針指向都會導致 APP 卡死？？

```
function hook_xxtea_decrypt() {
    Interceptor.attach(Module.findExportByName("libcocos2djs.so", "xxtea_decrypt"), {
        onEnter(args) {
            let jsc_data = args[0];
            let size = args[1].toInt32();
            let key = args[2].readCString();
            let key_len = args[3].toInt32();
            this.arg4 = args[4];

            let target_list = [0x15, 0x43, 0x73];
            let flag = true;
            for (let i = 0; i < target_list.length; i++) {
                if (target_list[i] != Memory.readU8(jsc_data.add(i))) {
                    flag = false;
                }
            }
            this.flag = flag;
            if (flag) {
                let new_size = size;
                let newAddress = Memory.alloc(new_size);
                Memory.protect(newAddress, new_size, "rwx")
                Memory.protect(args[0], new_size, "rwx")
                Memory.writeByteArray(newAddress, jsc_data.readByteArray(new_size))
                args[0] = newAddress;
            }

        },
        onLeave(retval) {
        }
    })

}
```

### 思路四實現

參考[這位大佬的文章](https://blog.shi1011.cn/rev/android/1422)可知 cocos2djs 內置的 v8 引擎最終通過`evalString`來執行`.jsc`解密後的 js 代碼。

在正式替換前，最好先通過 hook `evalString`的方式保存一份目標 js( 因為遊戲的熱更新策略等原因，可能導致`evalString`執行的 js 代碼與你從 apk 裡手動解密`.jsc`得到的 js 腳本有所不同 )。

```
function saveJscode(jscode, path) {
    var fopenPtr = Module.findExportByName("libc.so", "fopen");
    var fopen = new NativeFunction(fopenPtr, 'pointer', ['pointer', 'pointer']);
    var fclosePtr = Module.findExportByName("libc.so", "fclose");
    var fclose = new NativeFunction(fclosePtr, 'int', ['pointer']);
    var fseekPtr = Module.findExportByName("libc.so", "fseek");
    var fseek = new NativeFunction(fseekPtr, 'int', ['pointer', 'int', 'int']);
    var ftellPtr = Module.findExportByName("libc.so", "ftell");
    var ftell = new NativeFunction(ftellPtr, 'int', ['pointer']);
    var freadPtr = Module.findExportByName("libc.so", "fread");
    var fread = new NativeFunction(freadPtr, 'int', ['pointer', 'int', 'int', 'pointer']);
    var fwritePtr = Module.findExportByName("libc.so", "fwrite");
    var fwrite = new NativeFunction(fwritePtr, 'int', ['pointer', 'int', 'int', 'pointer']);

    let newPath = Memory.allocUtf8String(path);

    let openMode = Memory.allocUtf8String('w');

    let str = Memory.allocUtf8String(jscode);

    let file = fopen(newPath, openMode);
    if (file != null) {
        fwrite(str, jscode.length, 1, file)
        fclose(file);

    }
    return null;
}

function hook_evalString() {
    Interceptor.attach(Module.findExportByName("libcocos2djs.so", "_ZN2se12ScriptEngine10evalStringEPKclPNS_5ValueES2_"), {
        onEnter(args) {
            let path = args[4].readCString();
            path = path == null ? "" : path;
            let jscode = args[1];
            let size = args[2].toInt32();
            if (path.indexOf("assets/script/index.jsc") != -1) {
                saveJscode(jscode.readCString(), "/data/data/XXXXXXX/test.js");
            }
        }
    })
}
```

利用`Memory.scan`來找到修改的位置

```
function findReplaceAddr(startAddr, size, pattern) {
    Memory.scan(startAddr, size, pattern, {
        onMatch(address, size) {
            console.log("target offset: ", ptr(address - startAddr))
            return 'stop';
        },
        onComplete() {
            console.log('Memory.scan() complete');
        }
    });
}

function hook_evalString() {
    Interceptor.attach(Module.findExportByName("libcocos2djs.so", "_ZN2se12ScriptEngine10evalStringEPKclPNS_5ValueES2_"), {
        onEnter(args) {
            let path = args[4].readCString();
            path = path == null ? "" : path;
            let jscode = args[1];
            let size = args[2].toInt32();
            if (path.indexOf("assets/script/index.jsc") != -1) {
                let pattern = "76 61 72 20 65 20 3D 20 64 2E 50 6C 61 79 65 72 41 74 74 72 69 62 75 74 65 43 6F 6E 66 69 67 2E 67 65 74 44 72 65 61 6D 48 6C 70 65 49 74 65 6D 44 72 6F 70 28 29 2C";
                findReplaceAddr(jscode, size, pattern);
            }
        }
    })
}
```

最後以`Memory.writeU8`來逐字節修改，不用`Memory.writeUtf8String`的原因是它默認會在最終添加`'\0'`而導致報錯。

```
function replaceEvalString(jscode, offset, replaceStr) {
    for (let i = 0; i < replaceStr.length; i++) {
        Memory.writeU8(jscode.add(offset + i), replaceStr.charCodeAt(i))
    }
}

// 例:
function cheatAutoChopTree(jscode) {
    let replaceStr = 'true || "                                 "';
    replaceEvalString(jscode, 0x3861f6, replaceStr)
}
```

某砍樹手遊實踐
-------

以某款砍樹遊戲來進行簡單的實踐。

遊戲有自動砍樹的功能，但需要符合一定條件

![](https://bbs.kanxue.com/upload/attach/202409/946537_ES6S6JXU6F49CYT.webp)

如何找到對應的邏輯在哪個`.jsc`中？直接搜字符串就可以。

利用上述替換思路 4 來修改對應的 js 判斷邏輯，最終效果：

![](https://bbs.kanxue.com/upload/attach/202409/946537_UQX3R2EPVDQWWRQ.webp)

結語
--

思路 4 那種替換手段有大小限制，不能隨意地修改，暫時還未找到能隨意修改的手段，有知道的大佬還請不嗇賜教，有任何想法也歡迎交流 ^^

[[课程]FART 脱壳王！加量不加价！FART 作者讲授！](https://bbs.kanxue.com/thread-281194.htm)

最后于 13 小时前 被 ngiokweng 编辑 ，原因： no.

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#其他](forum-161-1-129.htm)