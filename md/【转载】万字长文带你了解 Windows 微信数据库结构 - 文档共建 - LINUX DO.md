> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [linux.do](https://linux.do/t/topic/1577333/1) ![](https://linux.do/user_avatar/linux.do/shenapex/96/504456_2.png)**原文发表于（已删除）：[https://blog.lc044.love/post/13](https://blog.lc044.love/post/13)**

转载说明

该文章原为 [GitHub - LC044/WeChatMsg](https://github.com/LC044/WeChatMsg) 项目的一部分，由作者 LC044 发布，详细解读了有关微信 3.0 和 4.0 数据库的结构，由于不可抗力因素，作者删除了该文章，但被引用的图片仍能访问，转载到 L 站是希望能留个档（也防止以后图片失效），方便其他开发者和用户查阅相关资料，同时发布在文档共建板块方便其他佬友补充修正  
**根据原作者要求，转载必须标明原出处**

正文删去了其余部分，仅保留了和数据库有关的内容，原文件在此

[万字长文带你了解 Windows 微信. zip](/uploads/short-url/b1odixleybZuXbjwvPZfnmJvt0J.zip) (772.1 KB)

感谢所有贡献者的付出

* * *

### 1.1 微信数据库目录结构

#### 1.1.1 微信 3

微信 3 数据库目录结构

```
text
├─Msg
│  │  Applet.db             # 小程序
│  │  BizChat.db            # 
│  │  BizChatMsg.db
│  │  ChatMsg.db
│  │  ChatRoomUser.db
│  │  ClientGeneral.db
│  │  CustomerService.db
│  │  Emotion.db            # 表情包
│  │  Favorite.db           # 微信收藏
│  │  FTSContact.db         # 搜索联系人
│  │  FTSFavorite.db        # 搜索收藏
│  │  FunctionMsg.db
│  │  HardLinkFile.db       # 微信文件
│  │  HardLinkImage.db      # 微信图片
│  │  HardLinkVideo.db      # 微信视频
│  │  ImageTranslate.db
│  │  LinkHistory.db
│  │  MicroMsg.db           # 微信联系人
│  │  Misc.db               # 微信联系人头像
│  │  MultiSearchChatMsg.db
│  │  NewTips.db
│  │  OpenIMContact.db      # 企业微信联系人
│  │  OpenIMMedia.db        # 企业微信语音
│  │  OpenIMMsg.db          # 企业微信聊天记录
│  │  OpenIMResource.db     # 企业微信联系人公司信息
│  │  PreDownload.db        
│  │  PublicMsg.db          # 公众号消息
│  │  PublicMsgMedia.db     # 公众号语音消息
│  │  Sns.db                # 朋友圈数据
│  │  StoreEmotion.db
│  │  SyncMsg.db
│  │  Voip.db
│  └─Multi                  # Multi文件夹里的数据库采用了分库操作，将一个大数据库分成多个便于提高检索效率
│          FTSMSG0.db       # 搜索聊天记录
│          FTSMSG1.db
│          MediaMSG0.db     # 语音消息 0
│          MediaMSG1.db     # 语音消息 1
│          MSG0.db          # 聊天记录 0
│          MSG1.db          # 聊天记录 1


```

#### 1.1.2 微信 4

微信 4 数据库目录结构

```
text
├─db_storage
│  │  
│  ├─biz
│  │      biz.db                
│  ├─contact                    # 联系人
│  │      contact.db
│  │      contact_fts.db
│  │      fmessage_new.db
│  │      wa_contact_new.db
│  ├─emoticon                   # 表情包
│  │      emoticon.db
│  ├─favorite                   # 微信收藏
│  │      favorite.db
│  │      favorite_fts.db
│  ├─hardlink                   # 微信图片、视频、文件
│  │      hardlink.db
│  ├─head_image                 # 微信头像
│  │      head_image.db
│  ├─ilinkvoip
│  │      ilinkvoip.db
│  ├─message                    # 聊天记录
│  │      biz_message_0.db      # 公众号消息
│  │      media_0.db            # 语音消息
│  │      message_0.db          # 聊天记录
│  │      message_1.db
│  │      message_fts.db
│  │      message_resource.db
│  │      message_revoke.db
│  ├─newtips
│  │      newtips.db
│  ├─session                    # 聊天会话窗口
│  │      session.db
│  ├─sns                        # 朋友圈
│  │      sns.db
│  ├─teenager
│  │      teenager.db
│  ├─tencentpay
│  │      tencentpay.db
│  ├─wcfinder
│  │      wcfinder.db
│  └─websearch
│          websearch.db


```

**Bash**

### 1.2 微信部分数据解析

#### 1.2.1 微信图片加密方式

**微信 3**

> 微信 4.0 之前的图片都采用了最简单的异或加密，即采用一个异或密钥（一个 0-255 的整数）与图片文件的每个字节进行异或运算。由于异或运算具有可逆性，采用相同的密钥再做一次异或运算即可得到原始图片文件。

**异或运算性质**

1.  交换律： A ^ B = B ^ A
2.  结合律： (A ^ B) ^ C = A ^ ( B ^ C )
3.  自反性： A ^ B ^ B = A

假设：

*   加密后数据：`C`
*   已知的原始文件头（Magic Number）：`P`
*   密钥：`K`
*   密钥长度：`L`
*   第 `i` 个字节：`i`

不同格式的图片通常有固定的文件头，例如：

*   **PNG**: `89 50 4E 47 0D 0A 1A 0A`
*   **JPG**: `FF D8 FF E0` 或 `FF D8 FF DB`
*   **BMP**: `42 4D`
*   **GIF**: `47 49 46 38 39 61` 或 `47 49 46 38 37 61`

根据异或运算的性质：

Ci=Pi⊕Ki

可得：

Ki=Ci⊕Pi

所以，我们可以通过已知的加密数据 `C` 和已知的文件头 `P` 计算密钥的前 `L` 个字节：

K0=C0⊕P0

K1=C1⊕P1

…

KL−1=CL−1⊕PL−1

密钥是一个 0-255 范围内的整数所以: key=K0=K1=K2⋅⋅⋅=KL−1

* * *

**微信 4**

> 微信 4 的图片采用 AES-EBC 和异或混合加密方式。

**.dat 文件头 (15 个字节)**

<table><thead><tr><th>大小 (字节)</th><th>内容 / 类型</th><th>说明</th></tr></thead><tbody><tr><td>6</td><td>0x07085631</td><td>.dat 文件标识符</td></tr><tr><td>4</td><td>int （小端序）</td><td>AES-EBC128 加密长度</td></tr><tr><td>4</td><td>int （小端序）</td><td>异或加密长度</td></tr><tr><td>1</td><td>0x01</td><td>未知</td></tr></tbody></table>

dat 文件头  

[![](https://linux.do/uploads/default/optimized/4X/c/2/f/c2f96f13f15c83f60b5da5776a0ea04ba3b11cf6_2_690x232.png)image999×336 74.7 KB](https://linux.do/uploads/default/original/4X/c/2/f/c2f96f13f15c83f60b5da5776a0ea04ba3b11cf6.png "image")

**文件末尾采用异或加密，加密长度最大为 1MB，多余部分未加密。**

> 例如：一个文件小于 1kB，则全部是 AES 加密，如果大于 1kB 且小于 1MB（实际是 1MB 零 1KB），则前 1KB 部分采用 AES 加密，剩余部分采用异或加密。如果文件大于 1MB，则前 1KB 采用 AES 加密，后面 1MB 采用异或加密，中间部分未加密。

dat 文件结构  

[![](https://linux.do/uploads/default/optimized/4X/c/f/3/cf3a9f148395d1b0f03c26d863230001681ca68a_2_690x459.jpeg)image1107×738 70.6 KB](https://linux.do/uploads/default/original/4X/c/f/3/cf3a9f148395d1b0f03c26d863230001681ca68a.jpeg "image")

*   AES 密钥为：`0xcfcd208495d565ef`
*   根据异或运算的可逆性，结合 jpg 文件末尾的 **FF D9** 两个字节，可以找一张微信生成的缩略图（一般为_t.dat 结尾），两次异或运算即可得到异或密钥。

#### 1.2.2 微信文件、文件夹命名规则

**微信 3**

```
text
├─Applet
├─Backup
├─FileStorage       # 存储微信图片、视频、文件
│  ├─Cache          # 缩略图缓存
│  ├─CustomEmotion  # 表情包
│  ├─Fav            # 收藏
│  ├─File           # 微信文件
│  ├─MsgAttach      # 微信里与每个人的聊天数据
│  │  └─aefd036248e0d4d0f07900a6239272e4    # 该联系人wxid的md5加密
│  │      ├─Image                           # 聊天图片
│  │      │  └─2024-10
│  │      ├─Thumb                           # 聊天中产生的缩略图
│  │      │  └─2024-10
│  │      ├─File                            # 合并转发的聊天记录里的文件
│  │      │  └─2024-10
│  │      └─Video                           # 合并转发的聊天记录里的视频
│  │          └─2024-10
│  ├─Sns            # 朋友圈数据
│  ├─Video          # 聊天视频
│  │  └─2025-02
│  └─XEditor
├─Msg               # 聊天记录数据库
└─ResUpdateV2


```

**Bash**

* * *

**微信 4**

```
text
├─business
├─cache             # 缓存
├─config            # 配置文件
├─db_storage        # 聊天记录数据库
├─msg               # 与好友的聊天数据
│  ├─attach         # 微信里与每个人的聊天数据
│  │  └─aefd036248e0d4d0f07900a6239272e4    # 该联系人wxid的md5加密
│  │  │  ├─2022-01                          # 日期
│  │  │  │  ├─Img                           # 聊天图片
│  │  │  │  └─Rec                           # 合并转发的聊天记录数据
│  │  │  │      └─2090_136022
│  │  │  │          └─Img
│  ├─file           # 微信文件
│  └─video          # 微信视频
├─Msg               # 聊天记录数据库
└─temp


```

**Bash**

微信 4 的文件命名方式跟之前有些微区别，但其数据存放是类似的（文件、视频单独存放在独立的文件夹中，图片按联系人分组存放到不同的文件夹中）

![](https://linux.do/images/emoji/twemoji/scroll.png?v=15)附录 1 微信 v4 数据库
------------------------------------------------------------------------

### 2.1 联系人 contact

> 这部分存储了微信里的联系人信息，包括好友、群聊、企业微信联系人、公众号等信息

#### **biz_info 公众号数据**

```
sql
CREATE TABLE biz_info(
    id INTEGER PRIMARY KEY, 
    username TEXT, 
    type INTEGER, 
    accept_type INTEGER, 
    child_type INTEGER, 
    version INTEGER, 
    external_info TEXT, 
    brand_info TEXT, 
    brand_icon_url TEXT, 
    brand_list TEXT, 
    brand_flag INTEGER, 
    belong TEXT, 
    ext_buffer BLOB
)


```

**[VB.Net](http://VB.Net)**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>id</td><td>INTEGER</td><td>跟 name2id 表的序号相对应，可以唯一确定一个联系人</td></tr><tr><td>username</td><td>TEXT</td><td>原始 wxid，可以唯一确定一个联系人</td></tr><tr><td>type</td><td>INTEGER</td><td>公众号类型，1 是公众号，0 是订阅号</td></tr><tr><td>accept_type</td><td>INTEGER</td><td></td></tr><tr><td>child_type</td><td>INTEGER</td><td></td></tr><tr><td>version</td><td>INTEGER</td><td></td></tr><tr><td>external_info</td><td>TEXT</td><td>存储了公众号的详细信息，公众号底部菜单信息</td></tr><tr><td>brand_info</td><td>TEXT</td><td></td></tr><tr><td>brand_icon_url</td><td>TEXT</td><td>logo 链接</td></tr><tr><td>brand_list</td><td>TEXT</td><td></td></tr><tr><td>brand_flag</td><td>INTEGER</td><td></td></tr><tr><td>belong</td><td>TEXT</td><td></td></tr><tr><td>ext_buffer</td><td>BLOB</td><td></td></tr></tbody></table>

* * *

#### **biz_session_feed 公众号消息会话**

```
sql
CREATE TABLE biz_session_feeds(
    username TEXT, 
    showname TEXT, 
    desc TEXT, type INTEGER, 
    unread_count INTEGER, 
    update_time INTEGER, 
    create_time INTEGER, 
    biz_attr_version INTEGER
)


```

**SQL**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>username</td><td>TEXT</td><td></td></tr><tr><td>showname</td><td>TEXT</td><td>屏幕上显示的名字</td></tr><tr><td>desc</td><td>TEXT</td><td>消息描述信息</td></tr><tr><td>type</td><td>INTEGER</td><td>消息类型</td></tr><tr><td>unread_count</td><td>INTEGER</td><td>未读消息的个数</td></tr><tr><td>update_time</td><td>INTEGER</td><td>更新时间</td></tr><tr><td>create_time</td><td>INTEGER</td><td>创建时间</td></tr><tr><td>biz_attr_version</td><td>INTEGER</td><td></td></tr></tbody></table>

* * *

#### **chat_room 群聊**

```
sql
CREATE TABLE chat_room(
  id INTEGER PRIMARY KEY, 
  username TEXT, 
  owner TEXT, 
  ext_buffer BLOB
)


```

**SQL**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>id</td><td>INTEGER</td><td>跟 name2id 表的序号相对应</td></tr><tr><td>username</td><td>TEXT</td><td>群聊的 username</td></tr><tr><td>owner</td><td>TEXT</td><td>群主的 username</td></tr><tr><td>ext_buffer</td><td>BLOB(protobuf)</td><td>存储了群成员的 username 和群昵称</td></tr></tbody></table>

```
python
syntax = "proto3";
package app.protobuf;
option go_package=".;proto";

message ChatRoomData {
  message ChatRoomMember {
      string wxID = 1;
      string displayName = 2;
      int32 state = 3;
  }
  repeated ChatRoomMember members = 1;
  int32 field_2 = 2;
  int32 field_3 = 3;
  int32 field_4 = 4;
  int32 room_capacity = 5;
  int32 field_6 = 6;
  int64 field_7 = 7;
  int64 field_8 = 8;
}


```

**Java**

* * *

#### **chat_room_info_detail 群聊信息**

```
sql
CREATE TABLE chat_room_info_detail(
    room_id_ INTEGER PRIMARY KEY, 
    username_ TEXT, 
    announcement_ TEXT, 
    announcement_editor_ TEXT, 
    announcement_publish_time_ INTEGER, 
    chat_room_status_ INTEGER, 
    room_top_msg_closed_id_list_text_ TEXT, 
    xml_announcement_ TEXT, 
    ext_buffer_ BLOB
)


```

**[VB.Net](http://VB.Net)**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>room_id_</td><td>INTEGER</td><td>跟 name2id 表的序号相对应</td></tr><tr><td>username_</td><td>TEXT</td><td>群聊的 username</td></tr><tr><td>announcement_</td><td>TEXT</td><td>存文本形式的群公告</td></tr><tr><td>announcement_editor_</td><td>TEXT</td><td>群功能编辑者的 username</td></tr><tr><td>announcement_publish_time_</td><td>INTEGER</td><td>发布时间</td></tr><tr><td>chat_room_status_</td><td>INTEGER</td><td></td></tr><tr><td>room_top_msg_closed_id_list_text_</td><td>TEXT</td><td></td></tr><tr><td>xml_announcement_</td><td>TEXT</td><td>xml 格式的群功能，可以解析出来更多信息，如图片、文件之类的信息</td></tr><tr><td>ext_buffer_</td><td></td><td></td></tr></tbody></table>

* * *

#### **chatroom_member 群成员表**

```
sql
CREATE TABLE chatroom_member(
    room_id INTEGER, 
    member_id INTEGER, 
    CONSTRAINT room_member UNIQUE(room_id, member_id)
)


```

**SQL**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>room_id</td><td>INTEGER</td><td>群聊 id，跟 name2id 表的序号相对应</td></tr><tr><td>member_id</td><td>INTEGER</td><td>群员 id，跟 name2id 表的序号相对应</td></tr></tbody></table>

* * *

#### **contact 联系人表**

> contact 存储了所有的联系人，包括好友、群聊、公众号、企业微信联系人

```
sql
CREATE TABLE contact(
    id INTEGER PRIMARY KEY, 
    username TEXT, 
    local_type INTEGER, 
    alias TEXT, 
    encrypt_username TEXT, 
    flag INTEGER, 
    delete_flag INTEGER, 
    verify_flag INTEGER, 
    remark TEXT, 
    remark_quan_pin TEXT, 
    remark_pin_yin_initial TEXT, 
    nick_name TEXT, 
    pin_yin_initial TEXT, 
    quan_pin TEXT, 
    big_head_url TEXT, 
    small_head_url TEXT, 
    head_img_md5 TEXT, 
    chat_room_notify INTEGER, 
    is_in_chat_room INTEGER, 
    description TEXT, 
    extra_buffer BLOB, 
    chat_room_type INTEGER
)


```

**[VB.Net](http://VB.Net)**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>id</td><td>INTEGER</td><td>序号，跟 name2id 表的序号相对应</td></tr><tr><td>user naI He</td><td>TEXT</td><td>联系人的 wixd</td></tr><tr><td>local_type</td><td>INTEGER</td><td>类型</td></tr><tr><td>alias</td><td>TEXT</td><td>微信里显示的微信号</td></tr><tr><td>encrypt_username</td><td>TEXT</td><td></td></tr><tr><td>flag</td><td>INTEGER</td><td>好像是联系人的真正类型，跟 3.0 数据库的 Type 对应</td></tr><tr><td>delete_flag</td><td>INTEGER</td><td></td></tr><tr><td>verify_flag</td><td>INTEGER</td><td></td></tr><tr><td>remark</td><td>TEXT</td><td>备注名</td></tr><tr><td>remark_quan_pin</td><td>TEXT</td><td>备注名全拼</td></tr><tr><td>remark_pin_yin_initial</td><td>TEXT</td><td>备注名拼音缩写</td></tr><tr><td>nick_name</td><td>TEXT</td><td>微信昵称</td></tr><tr><td>pin_yin_initial</td><td>TEXT</td><td>微信昵称拼音缩写</td></tr><tr><td>quan_pin</td><td>TEXT</td><td>微信昵称全拼</td></tr><tr><td>big_head_url</td><td>TEXT</td><td>好友头像大图</td></tr><tr><td>small_head_url</td><td>TEXT</td><td>好友头像小图</td></tr><tr><td>head_img_md5</td><td>TEXT</td><td>头像的 md5，可以通过 head_image.db 查询对应的头像</td></tr><tr><td>chat_room_notify</td><td>INTEGER</td><td>“chat_room_notify” INTEGER</td></tr><tr><td>is_in_chat_room</td><td>INTEGER</td><td>“is_in_chat_room” INTEGER</td></tr><tr><td>description</td><td>TEXT</td><td>“description” TEXT</td></tr><tr><td>extra_buffer</td><td>BLOB(protobuf)</td><td>存储了好友的详细信息，性别、地区、签名等</td></tr><tr><td>chat_room_type</td><td>INTEGER</td><td>“chat_room_type” INTEGER</td></tr></tbody></table>

相关信息

local_type 说明：

1：通讯录好友（包括公众号、手动添加到通讯录的群聊）

2：未添加到通讯录的群聊

3：群中的陌生人

5：企业微信好友

6：群聊中的陌生企业微信好友

相关信息

flag 说明：

flag 要转化为二进制，每一位代表不同的含义

第 7 位：代表是否是星标好友

第 12 位： 代表是否是置顶好友

第 17 位：代表是否屏蔽对方的朋友圈

第 24 位：代表是否是仅聊天好友

**`extra_buffer` 对应的 `.proto` 文件描述**

```
text
syntax = "proto3";

package example;

// 顶级消息定义
message ContactInfo {
  // varint 类型字段，根据数值范围选用 uint32 或 uint64
  uint32 gender  = 2;  // 性别:1 男 2:女 0:未知
  uint32 field3  = 3;
  string signature  = 4;  // 自助者天助！！！
  string country  = 5;  // CN
  string province  = 6;  // Shaanxi
  string city  = 7;  // Xi'an
  uint32 field8  = 8;
  string field9  = 9;
  uint32 field10 = 10; // 4294967295
  uint32 field11 = 11;
  uint32 field12 = 12;

  // 修改后的嵌套消息，对应 JSON 中 field 14 的数据结构
  MessageField14 phone_info = 14;

  string field15 = 15;
  uint32 field16 = 16;
  uint32 field17 = 17;
  uint32 field18 = 18;
  uint32 field19 = 19;
  string field20 = 20;
  string field21 = 21;
  uint32 field22 = 22;
  uint32 field23 = 23;
  uint32 field24 = 24;
  string field25 = 25;
  string field26 = 26;

  // 嵌套消息，朋友圈背景
  MessageField27 moments_info = 27;

  string field28 = 28;
  string field29 = 29;
  string label_list = 30;
  string field31 = 31;
  string field32 = 32;

  // 嵌套消息，对应 JSON 中 field 33 的 length_delimited 数据
  MessageField33 field33 = 33;

  string field34 = 34;
  string field35 = 35;
  MessageField36 field36 = 36;
  uint32 field37 = 37;
  uint32 field38 = 38; // 4294967295
}

// 定义 field14 对应的嵌套消息
// 修改后的嵌套消息，用于 field 14
message MessageField14 {
  uint32 field1 = 1; // varint 类型字段，存储数字
  repeated MessageField14_Result2 field2 = 2; // 这是一个 length_delimited 类型的字段，包含多个结果
}


message MessageField14_Result2 {
  string phone_numer = 1;  // string 类型字段，存储电话号码
}

// 定义 field27 对应的嵌套消息

message MessageField27 {
  uint32 field1 = 1;
  string background_url = 2; // 图片 URL
  uint64 field3 = 3; // 14588734692813845087（大数，用 uint64）
  uint32 field4 = 4; // 6785
  uint32 field5 = 5; // 4320
}

// 定义 field33 对应的嵌套消息

message MessageField33 {
  string field1 = 1;
}

message MessageField36 {
  MessageField36_Result results = 1;
}

message MessageField36_Result {
  string field1 = 1;
} 


```

**Java**

* * *

### 2.2 头像 head_image

**head_image 表**

```
sql
CREATE TABLE head_image(
    username TEXT PRIMARY KEY, 
    md5 TEXT, 
    image_buffer BLOB, 
    update_time INTEGER
)


```

**SQL**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>username</td><td>INTEGER</td><td>wxid</td></tr><tr><td>md5</td><td>TEXT</td><td>头像的 md5</td></tr><tr><td>image_buffer</td><td>BLOB</td><td>头像缩略图的二进制数据</td></tr><tr><td>update_time</td><td>INTEGER</td><td>更新时间</td></tr></tbody></table>

* * *

### 2.3 聊天记录 message

#### message_x.db 聊天数据

Msg_md5 表，每个联系人都单独建了一个表，命名规则：Msg_{md5(wxid)} , 对联系人的 wxid 用 md5 编码可以得到聊天记录所在的表

```
sql
CREATE TABLE Msg_xxxx(
    local_id INTEGER PRIMARY KEY AUTOINCREMENT, 
    server_id INTEGER, 
    local_type INTEGER, 
    sort_seq INTEGER, 
    real_sender_id INTEGER, 
    create_time INTEGER, 
    status INTEGER, 
    upload_status INTEGER, 
    download_status INTEGER, 
    server_seq INTEGER, 
    origin_source INTEGER, 
    source TEXT, 
    message_content TEXT, 
    compress_content TEXT, 
    packed_info_data BLOB, 
    WCDB_CT_message_content INTEGER DEFAULT NULL, 
    WCDB_CT_source INTEGER DEFAULT NULL
)


```

**SQL**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>local_id</td><td>INTEGER</td><td>自增 id</td></tr><tr><td>server_id</td><td>INTEGER</td><td>服务端的 id，每条消息的唯一 id</td></tr><tr><td>local_type</td><td>INTEGER</td><td>消息类型</td></tr><tr><td>sort_seq</td><td>INTEGER</td><td>用于排序的字段，时间戳 * 100，然后从 0 计数，如果某一秒发送了多条消息，可以通过该字段区分先后顺序</td></tr><tr><td>real_sender_id</td><td>INTEGER</td><td>发送者 id，可以通过 Name2Id 表获取实际的发送者 username</td></tr><tr><td>create_time</td><td>INTEGER</td><td>秒级时间戳</td></tr><tr><td>status</td><td>INTEGER</td><td>消息状态</td></tr><tr><td>upload_status</td><td>INTEGER</td><td>上传状态</td></tr><tr><td>download_status</td><td>INTEGER</td><td>下载状态</td></tr><tr><td>server_SEQ</td><td>INTEGER</td><td>服务端接收的顺序 id</td></tr><tr><td>origin_source</td><td>INTEGER</td><td></td></tr><tr><td>message_content</td><td>TEXT</td><td>消息的实际内容，local_type 是 1 时 message_content 是文本数据，其他类型都是 <code>Zstandard</code> 压缩后的 xml 二进制数据</td></tr><tr><td>compress_content</td><td>TEXT</td><td>压缩后的内容</td></tr><tr><td>packed_info_data</td><td>BLOB</td><td>protobuf 数据，存储了某些类型的聊天记录的信息（图片文件名、语音转文字记录、合并转发的聊天记录文件名）</td></tr><tr><td>WCD B_CT_message_content</td><td>INTEGER</td><td></td></tr><tr><td>WCD B_CT_source</td><td>INTEGER</td><td></td></tr></tbody></table>

**local_type 字段说明**

<table><thead><tr><th>local_type</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>1</td><td>文本</td><td></td></tr><tr><td>3</td><td>图片</td><td></td></tr><tr><td>34</td><td>语音</td><td></td></tr><tr><td>42</td><td>好友名片</td><td>包含好友名片和公众号名片</td></tr><tr><td>43</td><td>视频</td><td></td></tr><tr><td>47</td><td>表情包</td><td></td></tr><tr><td>48</td><td>发送的位置信息</td><td></td></tr><tr><td>50</td><td>音视频通话</td><td></td></tr><tr><td>66</td><td>企业微信好友名片</td><td></td></tr><tr><td>10000</td><td>系统消息</td><td>撤销信息，进群通知等</td></tr><tr><td>25769803825</td><td>文件</td><td></td></tr><tr><td>21474836529</td><td>分享链接</td><td></td></tr><tr><td>292057776177</td><td>分享链接</td><td></td></tr><tr><td>4294967345</td><td>分享链接</td><td></td></tr><tr><td>326417514545</td><td>分享链接</td><td></td></tr><tr><td>17179869233</td><td>分享链接</td><td></td></tr><tr><td>244813135921</td><td>引用消息</td><td></td></tr><tr><td>81604378673</td><td>合并转发的聊天记录</td><td></td></tr><tr><td>8594229559345</td><td>红包</td><td></td></tr><tr><td>219043332145</td><td>视频号</td><td></td></tr><tr><td>141733920817</td><td>小程序</td><td></td></tr><tr><td>154618822705</td><td>小程序</td><td></td></tr><tr><td>103079215153</td><td>转发的收藏笔记</td><td></td></tr><tr><td>266287972401</td><td>拍一拍</td><td></td></tr><tr><td>12884901937</td><td>音乐分享</td><td></td></tr></tbody></table>

**packed_info_data** 字段存储了某些聊天记录的附加信息，例如语音转文字，图片名（2025 年 3 月微信测试版修改了 img 命名方式才有了这个东西），合并转发的聊天记录文件夹名

*   图片
    
    ```
    text
    syntax = "proto3";
    // 2025年3月微信测试版修改了img命名方式才有了这个东西
    message PackedInfoDataImg {
      int32 field1 = 1;
      int32 field2 = 2;
      string filename = 3;
    }
    
    
    ```
    
    **Java**
    
*   语音
    
    ```
    text
    syntax = "proto3";
    
    package example;
    
    // 顶级消息定义
    message PackedInfoData {
      // varint 类型字段，根据数值范围选用 uint32 或 uint64
      uint32 field1  = 1;
      uint32 field2  = 2;
      MessageField5 info = 5;
    }
    
    // 定义 field14 对应的嵌套消息
    // 修改后的嵌套消息，用于 field 14
    message MessageField5 {
      uint32 field1 = 1;
      string audioTxt = 2; // 语音转文字结果
    }
    
    
    ```
    
    **Java**
    
*   合并转发的聊天记录
    
    ```
    text
      syntax = "proto3";
    
      message PackedInfoData {
        int32 field1 = 1;
        int32 field2 = 2;
        NestedMessage field7 = 7;
        AnotherNestedMessage info = 9;
      }
    
      message NestedMessage {
        SubMessage1 field1 = 1;
        SubMessage2 field2 = 2;
        string field3 = 3;
      }
    
      message SubMessage1 {
        int32 field1 = 1;
        string field2 = 2;
      }
    
      message SubMessage2 {
        string field1 = 1;
        string field2 = 2;
        string field3 = 3;
      }
    
      message AnotherNestedMessage {
        string dir = 1;
      }
    
    
    ```
    
    **Java**
    

#### biz_message_x.db 公众号消息

公众号消息记录，结构同 message_x.db

#### media_0.db 语音数据

```
sql
CREATE TABLE VoiceInfo(
  chat_name_id INTEGER, 
  create_time INTEGER, 
  local_id INTEGER, 
  svr_id INTEGER, 
  voice_data BLOB, 
  data_index TEXT DEFAULT '0'
)


```

**SQL**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>chat_name_id</td><td>INTEGER</td><td>结合 Name2Id 表可以查出发送者的 wxid</td></tr><tr><td>create_time</td><td>INTEGER</td><td>创建秒级时间戳</td></tr><tr><td>local_id</td><td>INTEGER</td><td>对应 message 数据库的 local_id</td></tr><tr><td>svr_id</td><td>INTEGER</td><td>对应 message 数据库的 server_id</td></tr><tr><td>voice_data</td><td>BLOB</td><td>语音 silk 的二进制数据</td></tr><tr><td>data_index</td><td>TEXT</td><td></td></tr></tbody></table>

* * *

### 2.4 文件、视频、图片 hardlink.db

**video_hardlink_info_v3**

```
sql
CREATE TABLE video_hardlink_info_v3(
    md5_hash INTEGER, 
    md5 TEXT, type INTEGER, 
    file_name TEXT, 
    file_size INTEGER, 
    modify_time INTEGER, 
    dir1 INTEGER, 
    dir2 INTEGER, 
    _rowid_ INTEGER PRIMARY KEY ASC, 
    extra_buffer BLOB
)


```

**SQL**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>md5hash</td><td>INTEGER</td><td></td></tr><tr><td>md5</td><td>TEXT</td><td>文件的 md5</td></tr><tr><td>type</td><td>INTEGER</td><td>文件类型：3：正常的文件，5：合并转发的聊天记录里的文件</td></tr><tr><td>file_name</td><td>INTEGER</td><td>文件名</td></tr><tr><td>file_size</td><td>INTEGER</td><td>文件大小 (B)</td></tr><tr><td>modeify_time</td><td>INTEGER</td><td>修改时间</td></tr><tr><td>dir1</td><td>INTEGER</td><td>文件夹 1 的 id，dir2id 表对应第 id 行 usernamez 字段为 dir1 的文件夹名</td></tr><tr><td>dir2</td><td>INTEGER</td><td>文件夹 2 的 id，type 为 3 时 dir2 值为 0，只需要从 msg/video/dir1 文件夹中找就行了</td></tr><tr><td>_<em>rowid</em></td><td>INTEGER</td><td></td></tr><tr><td>extra_buffer</td><td>BLOB</td><td>protocbuf 数据，存储另一个文件夹 dir3，type 为 5 时有效。.proto 文件描述：<code>syntax = "proto3"; package example; message FileInfoData { string dir3 = 1; uint32 file_size = 2; }</code></td></tr></tbody></table>

```
sql
select file_size,type,file_name,dir2id.username,_rowid_,modify_time
from video_hardlink_info_v3
join  dir2id on dir2id.rowid = dir1
where md5=10086;


```

**C#**

合并转发的聊天记录

*   文件路径：
    *   msg\attach\9e20f478899dc29eb19741386f9343c8\2025-03\Rec\409af365664e0c0d\F\5\xxx.pdf
*   图片路径：
    *   msg\attach\9e20f478899dc29eb19741386f9343c8\2025-03\Rec\409af365664e0c0d\Img\5
*   视频路径：
    *   msg\attach\9e20f478899dc29eb19741386f9343c8\2025-03\Rec\409af365664e0c0d\V\5.mp4

```
9e20f478899dc29eb19741386f9343c8` 是wxid的md5加密，`409af365664e0c0d` 是上述 `extra_buffer` 字段里的 `dir3


```

文件夹最后的 5 代表的该文件是合并转发的聊天记录第 5 条消息，如果存在嵌套的合并转发的聊天记录，则依次递归的添加上一层的文件名后缀，例如：合并转发的聊天记录有两层

```
plain
0：文件（文件夹名为0）

1：图片 （文件名为1）

2：合并转发的聊天记录

    0：文件（文件夹名为2_0）

    1：图片（文件名为2_1）

    2：视频（文件名为2_2.mp4）


```

**Markdown**

```
python
def parser_merged(merged_messages, level):
    for index, inner_msg in enumerate(merged_messages):
        wxid_md5 = hashlib.md5(username.encode("utf-8")).hexdigest()
        if inner_msg.type == MessageType.Image:
           inner_msg.path = os.path.join(
               'msg', 'attach', wxid_md5, month,
                'Rec', dir0, 'Img', f"{level}{'_' if level else ''}{index}")
           inner_msg.thumb_path = os.path.join(
               'msg', 'attach', wxid_md5, month,
                'Rec', dir0, 'Img', f"{level}{'_' if level else ''}{index}_t")
       elif inner_msg.type == MessageType.Video:
            inner_msg.path = os.path.join(
                'msg', 'attach',wxid_md5,month,
                'Rec', dir0, 'V', f"{level}{'_' if level else ''}{index}.mp4")
        elif inner_msg.type == MessageType.File:
            inner_msg.path = os.path.join(
                'msg', 'attach',wxid_md5,month,
                'Rec', dir0, 'F', f"{level}{'_' if level else ''}{index}", 
                inner_msg.file_name)
        elif inner_msg.type == MessageType.MergedMessages:
            parser_merged(
                inner_msg.messages, 
                f'{index}' if not level else f'{level}_{index}'
            )

parser_merged(msg.messages, '')


```

**Python**

* * *

### 2.2 会话窗口 session

session.db 存储了微信聊天界面显示的会话窗口信息

```
sql
CREATE TABLE SessionTable(
  username TEXT PRIMARY KEY, 
  type INTEGER, 
  unread_count INTEGER, 
  unread_first_msg_srv_id INTEGER, 
  is_hidden INTEGER, 
  summary TEXT, 
  draft TEXT, 
  status INTEGER, 
  last_timestamp INTEGER, 
  sort_timestamp INTEGER, 
  last_clear_unread_timestamp INTEGER, 
  last_msg_locald_id INTEGER, 
  last_msg_type INTEGER, 
  last_msg_sub_type INTEGER, 
  last_msg_sender TEXT, 
  last_sender_display_name TEXT, 
  last_msg_ext_type INTEGER
)


```

**[VB.Net](http://VB.Net)**

### 2.3 表情包 emoticon

emoticon.db 存储了用户收藏的表情包信息

```
sql
CREATE TABLE kNonStoreEmoticonTable(
  type INTEGER, 
  md5 TEXT, 
  caption TEXT, 
  product_id TEXT, 
  aes_key TEXT, 
  thumb_url TEXT, 
  tp_url TEXT, 
  auth_key TEXT, 
  cdn_url TEXT, 
  extern_url TEXT, 
  extern_md5 TEXT, 
  encrypt_url TEXT
)


```

**[VB.Net](http://VB.Net)**

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>type</td><td>INTEGER</td><td></td></tr><tr><td>md5</td><td>TEXT</td><td>表情包的 md5，可以从 message 数据库里获取</td></tr><tr><td>caption</td><td>TEXT</td><td></td></tr><tr><td>product_id</td><td>TEXT</td><td></td></tr><tr><td>aes_key</td><td>TEXT</td><td></td></tr><tr><td>thumb_url</td><td>TEXT</td><td>表情包缩略图 url（可以直接引用）</td></tr><tr><td>cdn_url</td><td>TEXT</td><td>表情包 url（可以直接引用）</td></tr><tr><td>extern_url</td><td>TEXT</td><td></td></tr><tr><td>extern_md5</td><td>TEXT</td><td></td></tr><tr><td>encrypt_url</td><td>TEXT</td><td></td></tr></tbody></table>

* * *

### 2.4 朋友圈 sns

**SnsTimeLine 表存储了每条朋友圈的详细信息**

```
sql
CREATE TABLE SnsTimeLine(
  tid INTEGER PRIMARY KEY DESC, 
  user_name TEXT, 
  content TEXT
)


```