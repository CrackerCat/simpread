> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268526.htm)

> cocos2d 逆向入门和某捕鱼游戏分析

目录

*   [初步认识 cocos2d-x](#初步认识cocos2d-x)
*   [从 c++ 进入 lua 世界](#从c++进入lua世界)
*            [applicationDidFinishLaunching 函数](#applicationdidfinishlaunching函数)
*            LuaStack::executeScriptFile
*            LuaStack::luaLoadBuffer
*   [总结](#总结)
*   [实战](#实战)
*   [个人想法](#个人想法)
*            [参考文章](#参考文章)

初步认识 cocos2d-x
==============

先 clone 到本地

 

git clone https://github.com/cocos2d/cocos2d-x.git

 

Cocos2d-x 是一个开源的移动 2D 游戏框架，底层支持各种平台，核心用 c++ 封装了各种库，外面给了 lua 和 c++ 的接口，所以关键代码可能在 lua 中，很多安卓游戏的逻辑也基本都在 lua 脚本里，盗用官网这张图

 

![](https://bbs.pediy.com/upload/attach/202107/877073_8S5R433RFN2RNT7.png)

从 c++ 进入 lua 世界
===============

lua 虚拟机相关代码在 cocos2d-x\cocos\scripting\lua-bindings\manual 里

 

CCLuaEngine.h lua 引擎相关

 

CCLuaStack.h lua 栈相关

 

进入虚拟机

 

cocos2d-x\templates\lua-template-default\frameworks\runtime-src\Classes\AppDelegate.cpp\AppDelegate::applicationDidFinishLaunching

applicationDidFinishLaunching 函数
--------------------------------

应用结束加载中进入 lua 虚拟机

```
bool AppDelegate::applicationDidFinishLaunching()
{
    // set default FPS
    Director::getInstance()->setAnimationInterval(1.0 / 60.0f);//设置fps刷新率
 
    // register lua module
    auto engine = LuaEngine::getInstance();//创建lua虚拟机引擎
    ScriptEngineManager::getInstance()->setScriptEngine(engine);//设置脚本引擎为lua引擎
    lua_State* L = engine->getLuaStack()->getLuaState();//创建lua虚拟机环境lua_State
    lua_module_register(L);  //分配网络，控制台，ui界面等相关联的寄存器
 
    register_all_packages();
 
    LuaStack* stack = engine->getLuaStack();
    stack->setXXTEAKeyAndSign("2dxLua", strlen("2dxLua"), "XXTEA", strlen("XXTEA"));
    //这一步很关键，获取栈结构并调用setXXTEAKeyAndSign设置加密算法为xxtea，sign为XXTEA,KEY为2dxlua，很多游戏lua脚本都用的默认的sign和key
    //如果没有用默认的，ida打开libxxxluaxxx.so直接搜索applicationDidFinishLaunching导出函数也基本都能直接找到
 
    //register custom function
    //LuaStack* stack = engine->getLuaStack();
    //register_custom_function(stack->getLuaState());
 
#if CC_64BITS
    FileUtils::getInstance()->addSearchPath("src/64bit");
#endif
    FileUtils::getInstance()->addSearchPath("src");  //lua源码在src文件夹，资源在res文件夹
    FileUtils::getInstance()->addSearchPath("res");
    if (engine->executeScriptFile("main.lua"))   //直接通过lua引擎调用main.lua进入lua的世界
    {
        return false;
    }
 
    return true;
}

```

这句 engine->executeScriptFile("main.lua") 调用了 cocos2d-x\cocos\scripting\lua-bindings\manual\CCLuaEngine.cpp 的 executeScriptFile 调用了 CCLuaStack.cpp 的 executeScriptFile

LuaStack::executeScriptFile
---------------------------

```
int LuaStack::executeScriptFile(const char* filename)
{
    CCAssert(filename, "CCLuaStack::executeScriptFile() - invalid filename");
 
    std::string buf(filename);
    //
    // remove .lua or .luac
    //
    size_t pos = buf.rfind(BYTECODE_FILE_EXT);//BYTECODE_FILE_EXT就是lua字节码，NOT_BYTECODE_FILE_EXT就是lua脚本源码
    //static const std::string BYTECODE_FILE_EXT    = ".luac";
    if (pos != std::string::npos)
    {
        buf = buf.substr(0, pos);//截取前缀
    }
    else
    {
        pos = buf.rfind(NOT_BYTECODE_FILE_EXT);
        if (pos == buf.length() - NOT_BYTECODE_FILE_EXT.length())
        {
            buf = buf.substr(0, pos);
        }
    }
 
    FileUtils *utils = FileUtils::getInstance();
 
    //
    // 1. check .luac suffix
    // 2. check .lua suffix
    //
    std::string tmpfilename = buf + BYTECODE_FILE_EXT;
    if (utils->isFileExist(tmpfilename))
    {
        buf = tmpfilename;
    }
    else
    {
        tmpfilename = buf + NOT_BYTECODE_FILE_EXT;
        if (utils->isFileExist(tmpfilename))
        {
            buf = tmpfilename;
        }
    }
 
    std::string fullPath = utils->fullPathForFilename(buf);//获取绝对路径
    Data data = utils->getDataFromFile(fullPath);//通过getDataFromFile读取lua文件到data
    int rn = 0;
    if (!data.isNull())
    {
        if (luaLoadBuffer(_state, (const char*)data.getBytes(), (int)data.getSize(), fullPath.c_str()) == 0)//通过luaLoadBuffer加载data
        {
            rn = executeFunction(0);
        }
    }
    return rn;
}

```

LuaStack::luaLoadBuffer
-----------------------

luaLoadBuffer 里调用 xxtea_decrypt 解密了 lua 脚本，然后调用 luaL_loadbuffer 加载解密后的脚本，所以直接 hook 这个函数 luaL_loadbuffer 把 (char*)content 这个字符 dump 出来就得到解密过的 lua 脚本了

```
int LuaStack::luaLoadBuffer(lua_State *L, const char *chunk, int chunkSize, const char *chunkName)
{
    int r = 0;
 
    if (_xxteaEnabled && strncmp(chunk, _xxteaSign, _xxteaSignLen) == 0)//这里判断是否开启xxtea加密，如果开启就需要解密
    {
        // decrypt XXTEA
        xxtea_long len = 0;
        unsigned char* result = xxtea_decrypt((unsigned char*)chunk + _xxteaSignLen,
                                              (xxtea_long)chunkSize - _xxteaSignLen,
                                              (unsigned char*)_xxteaKey,
                                              (xxtea_long)_xxteaKeyLen,
                                              &len);//调用xxtea_decrypt解密脚本，这个函数在cocos2d-x\external\xxtea\xxtea.cpp里，加解密都在这个cpp里
        unsigned char* content = result;
        xxtea_long contentSize = len;
        skipBOM((const char*&)content, (int&)contentSize);//忽略utf8的bom
        r = luaL_loadbuffer(L, (char*)content, contentSize, chunkName);//无论是否加密，解密后都会调用luaL_loadbuffer函数，所以直接hook这个函数把(char*)content这个字符dump出来就是解密过的lua脚本了
        free(result);
    }
    else
    {
        skipBOM(chunk, chunkSize);
        r = luaL_loadbuffer(L, chunk, chunkSize, chunkName);//这个返回值r会反映加载失败的类型，在下面的switch中打印出来
    }
 
#if defined(COCOS2D_DEBUG) && COCOS2D_DEBUG > 0
    if (r)
    {
        switch (r)
        {
            case LUA_ERRSYNTAX:
                CCLOG("[LUA ERROR] load \"%s\", error: syntax error during pre-compilation.", chunkName);
                break;
 
            case LUA_ERRMEM:
                CCLOG("[LUA ERROR] load \"%s\", error: memory allocation error.", chunkName);
                break;
 
            case LUA_ERRFILE:
                CCLOG("[LUA ERROR] load \"%s\", error: cannot open/read file.", chunkName);
                break;
 
            default:
                CCLOG("[LUA ERROR] load \"%s\", error: unknown.", chunkName);
        }
    }
#endif
    return r;
}

```

而 luaL_loadbuffer 的源码没有，只有编译过的库 cocos2d-x\external\lua\luajit\prebuilt\android\armeabi-v7a\libluajit.a，

 

要找它的实现需要下载 luajit 源代码分析了，这就完全进入了 lua 虚拟机的实现

 

a **Just-In-Time Compiler** for Lua. **采用 C 语言写的 Lua 的解释器的代码**

总结
==

1. 从 c++ 进入 lua 世界的调用逻辑

```
AppDelegate::applicationDidFinishLaunching
 
{
 
  setXXTEAKeyAndSign
 
  executeScriptFile
 
  {
 
​    getDataFromFile
 
​    luaLoadBuffer
 
​    {
 
​      xxtea_decrypt
 
​      luaL_loadbuffer
 
​      {
 
​        luajit
 
​       }
 
​     }
 
​    executeFunction
 
  }
 
}

```

2. 加密算法为 xxtea，如果没有修改，sign 为 XXTEA,KEY 为 2dxlua，如果有修改，可以 ida 打开 libxxluaxx.so 在 applicationDidFinishLaunching 里找到

 

3. 无论是否加密，解密后都会调用 luaL_loadbuffer 函数，所以直接 hook 这个函数把 (char*)content 这个字符 dump 出来就是解密过的 lua 脚本了，缺点是要把游戏运行一遍，只能搞出执行过的代码

 

4.cocos2d-x\external\xxtea\xxtea.cpp 里有完整的加密解密算法，逻辑清晰，可以写个 python 脚本直接本地解密，也可以在这里 hook 获取 key 和 sign 或者解密后脚本

实战
==

某捕鱼游戏，下载安装 apk 后，再其内部内置了捕鱼、麻将等十几款小游戏，嘿嘿，懂得都懂这是干啥的，直接点击捕鱼游戏下载，

 

下载后的游戏源码在 / data/data/com.q8wdw6.gyll9spfo.nycatp9/files/download / 里

 

adb pull 出来，files 文件夹里面已经暴露很多信息了，配置，下载的游戏，更新等等

 

![](https://bbs.pediy.com/upload/attach/202107/877073_38GTBGTYV7U4YPN.png)

 

进入 files\download\107\res \ 就可以看到 luac，很明显 crash 里面存储了所有碰撞，model_path_crash 和 path 里存储的是移动路径，我们可以看到所有的鱼的移动路径都是 tm 设定好的，models 里面存储着游戏逻辑，views 里面存储着界面显示逻辑

 

![](https://bbs.pediy.com/upload/attach/202107/877073_JXFQE8PMCCXRCEE.png)

 

随机打开一个 luac 加密了，开头的 ZX_CS@56#D~d@dud 明显就是加密 sign

 

![](https://bbs.pediy.com/upload/attach/202107/877073_3R4FFSM95AXHN7B.png)

 

根据之前对 cocos2d 引擎的分析，找到 apk 下面带有 lua 也是最大的一个 so libqpry_lua.so，用 ida 直接打开，定位到 AppDelegate::applicationDidFinishLaunching 函数，直接拖到最下面，看到这句

 

(_*(v7 + 1) + 116))(_(v7 + 1), "ZX_01RdsF~@!R8", 14, "ZX_CS@56#D~d@dud", 16); 很明显是调用了 stack->setXXTEAKeyAndSign

 

对比源码 stack->setXXTEAKeyAndSign("2dxLua", strlen("2dxLua"), "XXTEA", strlen("XXTEA"));

 

解密 key 为 ZX_01RdsF~@!R8，sign 为 ZX_CS@56#D~d@dud

 

也可以直接搜索字符串，因为 key 和 sign 在一个函数调用，所以一般离的很近，base/src/main.lua 这个字符串进一步验证了正确性

 

![](https://bbs.pediy.com/upload/attach/202107/877073_2CNR8BXEVKJ5A37.png)

```
int __fastcall AppDelegate::applicationDidFinishLaunching(AppDelegate *this)`
`{`
 
//`前面是一堆初始化啥的没用的`
 
....
 
`(**(v7 + 1) + 116))(*(v7 + 1), "ZX_01RdsF~@!R8", 14, "ZX_CS@56#D~d@dud", 16);`
  `if ( (*(*v7 + 28))(v7, "base/src/main.lua") )`
    `return 0;`
  `v26 = GetMCKernel();`
  `if ( !v26 )`
    `return 0;`
  `v27 = *(this + 145);
  if ( v27 )
    v27 += 580;
  v28 = (*(*v26 + 20))(v26, v27);`
  `v29 = *(cocos2d::Director::getInstance(v28) + 152);`
  `v32[0] = AppDelegate::GlobalUpdate;`
  `v32[1] = -8;`
  `cocos2d::Scheduler::schedule(v29, AppDelegate::GlobalUpdate, -8, this + 4, 0.0, 0xFFFFFFFE, 0.0, 0);`
  `return v25;`
`}`

```

有了 key 和 sign 就可以直接解密 luac 脚本了，照着写，很简单，可以看到 sign 的作用就是被忽略，只有长度有用，尴尬，解密用的主要是 key

```
unsigned char* result = xxtea_decrypt((unsigned char*)chunk + _xxteaSignLen,
                                          (xxtea_long)chunkSize - _xxteaSignLen,
                                          (unsigned char*)_xxteaKey,
                                          (xxtea_long)_xxteaKeyLen,
                                          &len);
 
unsigned char *xxtea_decrypt(unsigned char *data, xxtea_long data_len, unsigned char *key, xxtea_long key_len, xxtea_long *ret_length)
{
    unsigned char *result;
 
    *ret_length = 0;
 
    if (key_len < 16) {
        unsigned char *key2 = fix_key_length(key, key_len);
        result = do_xxtea_decrypt(data, data_len, key2, ret_length);
        free(key2);
    }
    else
    {
        result = do_xxtea_decrypt(data, data_len, key, ret_length);
    }
 
    return result;
}

```

或者直接 pip install xxtea-py

 

然后直接 python，自己写个循环实现批量解密吧

```
import xxtea
import os
   orig_path=“”//初始路径
   new_path=“”//存储路径
   xxtea_sign=“”
   xxtea_key=“”
   orig_file = open(orig_path, "rb")
   encrypt_bytes = orig_file.read()
   orig_file.close()
   decrypt_bytes = xxtea.decrypt(encrypt_bytes[len(xxtea_sign):], xxtea_key)
   new_file = open(new_path, "wb")
   new_file.write(decrypt_bytes)
   new_file.close()

```

下面开始分析 lua，这个最简单，我们想分析子弹打到鱼的逻辑，直接找 src\views\layer\BulletLayer.luac

 

直接找子弹打到鱼的函数，很清晰，第一个参数为子弹对象，第二个为鱼列表，这里看到只有 master_id == self_player_id 才会调用

 

on_self_bullet_crash_fish 当自己的子弹击中了鱼最后会调用加金币等，否则只调用 on_bullet_crash_fish 显示效果，这里可以也改成 on_self_bullet_crash_fish 就可以加金币了

```
function BulletLayer:on_bullet_crash_fish(bullet_obj, t_fish_list)
    local scene = self:get_scene()
    local master_id = bullet_obj:get_master_id()
    local bullet_uid = bullet_obj:get_bullet_uid()
    local self_player_id = self:get_self_player_id()
    local num = #t_fish_list
    if(num >= 2) then
        local fish_layer = self:get_fish_layer()
        table.sort(t_fish_list, function(uid1, uid2)
        local fish_obj1 = fish_layer:get_fish_by_uid(uid1);local fish_obj2 = fish_layer:get_fish_by_uid(uid2)
            local zorder1 = fish_obj1:getLocalZOrder();local zorder2 = fish_obj2:getLocalZOrder()
            if(zorder1 > zorder2) then return true end
            end)
    end
    local fish_uid = t_fish_list[1]
    if(master_id == self_player_id) then
        scene:on_self_bullet_crash_fish(master_id, bullet_uid, fish_uid)
        --scene:on_self_bullet_crash_fish_test(master_id, bullet_uid, fish_uid)
    end
    scene:on_bullet_crash_fish(master_id, bullet_uid, t_fish_list)
end

```

在定位一下加钱的函数，fish_gold 是鱼的钱，fish_odds 是鱼的剩余 ，这句 local one_add_gold = math.floor(fish_gold/12) 就是一个鱼加的钱，我们把这个 / 12 改为 * 12 就可以修改倍数了

```
function AnimationLayer:play_catch_fish_earn_money_self(fish_gold, fish_odds)
    local earn_money_node = self:reuse_new_earn_money_view_node('zhuanqianle_buyu', self, 0)
    local pos = cc.p(sizeVisble.width/2, sizeVisble.height/2)
    local one_add_gold = math.floor(fish_gold/12)
    local min = self:get_earn_money_one_add_gold_min()
    if(one_add_gold < min) then one_add_gold = min end
    earn_money_node:setPosition(pos);  earn_money_node:set_gain_gold(fish_gold)
    earn_money_node:set_one_add_gold(one_add_gold); earn_money_node:set_cur_gold(0)
    local delay_time = cc.DelayTime:create(4)
    local call_back = cc.CallFunc:create(handler(self, self.call_back_reuse_obj))
    local seq = cc.Sequence:create(delay_time, call_back);
    earn_money_node:setScale(1)
    earn_money_node:runAction(seq);
    earn_money_node:play_ani()
end

```

入口代码在 AnimationLayer.lua 里，直接看接收到玩家捕到鱼的函数，分为网或者炸弹，里面还有一网打尽和大转盘之类的特效，这里可以把它全部改成高倍的特效

```
function AnimationLayer:on_recv_player_catch_fish(player_id, fish_uid, fish_gold)
    local fish_layer = self:get_fish_layer()
    local fish_obj = fish_layer:get_fish_by_uid(fish_uid)
    if(fish_obj == nil) then return end 
    local data_center = self:get_data_center()
    local fish_id = fish_obj:get_fish_id()
    local fish_info = data_center:get_fish_info(fish_id)
    if(fish_info == nil) then return end
    local fish_type = fish_info.type
    if(fish_type == GameDefine.fish_type.wang) then
        local other_fish_obj_list = self:get_other_fish_yi_wang_da_jin_list(fish_obj, GameDefine.fish_type.wang)
        local num = #other_fish_obj_list
        if(num > 0) then
            local fish_odds = self:get_catch_fish_odds(fish_obj, other_fish_obj_list)
            self:on_player_catch_fish_yi_wang_da_jin(player_id, fish_obj, fish_gold, fish_odds, other_fish_obj_list)
        end
        self:on_player_catch_fish_drop_fish_gold(player_id, fish_obj, 0, 0)
        return
    end
    if(fish_type == GameDefine.fish_type.bomb) then
        play_drop_gold = 0
        local other_fish_list = self:get_bomb_fish_effect_fish_list()
        local num = #other_fish_list
        if(num > 0) then
           local fish_odds = self:get_catch_fish_odds(fish_obj, other_fish_list)
            self:on_player_catch_fish_bomb(player_id, fish_obj, fish_gold, fish_odds, other_fish_list)
        end
        self:on_player_catch_fish_drop_fish_gold(player_id, fish_obj, 0, 0)
        return
    end
    local fish_odds = fish_info.mulriple_max;local fish_name = fish_info.name
    self:on_player_cath_fish_da_zhuan_pan(player_id, fish_type, fish_gold, fish_name)
    self:on_player_catch_fish_drop_fish_gold(player_id, fish_obj, fish_gold, fish_odds)
end

```

src\models\DataCenter.lua 里存储着鱼和子弹等参数，可以看到子弹最高 9 级，odds_min 与 odds_max 应该就代表威力，改这里可以改子弹威力

```
{
   room_style= 3,
   cannon_id= 109,
   odds_min= 7,
   odds_max= 10,
   level= 3,
   res= "pao3_buyu",
   bullet= "zd9",
   net= "yuwang3_buyu",
   time= 200,
   sound= 109
}

```

self.fish_client 存储不同鱼的参数，不同 id 对应不同鱼，mulriple_min 与 mulriple_max 存储着金币的倍数最大与最小值，一般鱼都是 2-3 倍，大章鱼为 300 倍，改这里可以直接改鱼的倍数

```
{
   id= 302,
   ,
   packet= "4lian_buyu",
   crash_model= 101,
   mulriple_min= 60,
   mulriple_max= 60,
   type= 101,
   zoder= 21,
   die_sound= "fish14_1",
   die_type= 4,
   copy_num= 4
},
{
   id= 402,
   ,
   packet= "yu22_buyu",
   crash_model= 16,
   mulriple_min= 300,
   mulriple_max= 300,
   type= 2,
   zoder= 30,
   bron_sound= "f_wb_3",
   die_sound= "fish33_1",
   die_type= 5,
   copy_num= 4
},

```

其他的功能逻辑获得源码后都很清晰，改倍数改鱼改子弹什么都很简单，改完之后重新用 sign 和 key 加密之后 push 到相应文件夹下面就实现了破解

个人想法
====

如何实现 cocos2d 反逆向，我的一些不成熟想法，由浅到深分层如下

 

1. 修改 xxtea 的 key 和 sign，就像这个游戏，需要分析 so 才能找到 key

 

2. 直接修改 xxtea 算法为其他可逆算法，这样逆向的时候还得逆加密算法

 

3. 在进一步，修改 luajit 源码改动 lua 虚拟机，比如改变字节码指令顺序，或者改变数据读取顺序等，这样逆向的时候必须分析 lua 虚拟机的改动

 

4. 把 key，加密算法，虚拟机改动等的关键函数封装放到其他 cpp 或者其他 so 里，在加密一层，调用的时候在解密，这样需要先动态调试先分析调用逻辑

 

5. 关键代码加入 ollvm 再编译或者直接 vm 掉，这样需要先去掉 ollvm 混淆或者分析 vm

参考文章
----

[cocos2d 3.3 lua 代码加密 luac](https://www.cnblogs.com/lxjshuju/p/7028393.html)

 

[安卓逆向之 Luac 解密反编译](https://www.yuanrenxue.com/app-crawl/luac.html)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 5 天前 被挤蹭菌衣编辑 ，原因： 图挂了

[#其他](forum-161-1-129.htm)