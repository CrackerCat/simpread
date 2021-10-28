> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270014.htm)

> [原创] 在 lineageOS16.0 小米 6 上 复现无人直播应用

这是今年五月份时出于好奇研究了一翻. 具体原理我也不解释了. 直接贴代码.  
附件中有所有用到的文件

```
project frameworks/av/
diff --git a/services/audiopolicy/service/AudioPolicyInterfaceImpl.cpp b/services/audiopolicy/service/AudioPolicyInterfaceImpl.cpp
index 0a4e66566..40b194c4f 100644
--- a/services/audiopolicy/service/AudioPolicyInterfaceImpl.cpp
+++ b/services/audiopolicy/service/AudioPolicyInterfaceImpl.cpp
@@ -356,7 +356,7 @@ status_t AudioPolicyService::getInputForAttr(const audio_attributes_t *attr,
     }
 
     // check calling permissions
-    if (!isTrustedCallingUid(callingUid) && !recordingAllowed(opPackageName, pid, uid)) {
+   /* if (!isTrustedCallingUid(callingUid) && !recordingAllowed(opPackageName, pid, uid)) {
         ALOGE("%s permission denied: recording not allowed for uid %d pid %d",
                 __func__, uid, pid);
         return PERMISSION_DENIED;
@@ -367,7 +367,7 @@ status_t AudioPolicyService::getInputForAttr(const audio_attributes_t *attr,
         attr->source == AUDIO_SOURCE_VOICE_CALL) &&
         !captureAudioOutputAllowed(pid, uid)) {
         return PERMISSION_DENIED;
-    }
+    }*/
 
     if ((attr->source == AUDIO_SOURCE_HOTWORD) && !captureHotwordAllowed(pid, uid)) {
         return BAD_VALUE;
@@ -398,8 +398,9 @@ status_t AudioPolicyService::getInputForAttr(const audio_attributes_t *attr,
                 // FIXME: use the same permission as for remote submix for now.
             case AudioPolicyInterface::API_INPUT_MIX_CAPTURE:
                 if (!isTrustedCallingUid(callingUid) && !captureAudioOutputAllowed(pid, uid)) {
-                    ALOGE("getInputForAttr() permission denied: capture not allowed");
-                    status = PERMISSION_DENIED;
+                    ALOGE("getInputForAttr() permission allowed: capture allowed");
+                    //ALOGE("getInputForAttr() permission denied: capture not allowed");
+                    //status = PERMISSION_DENIED;
                 }
                 break;
             case AudioPolicyInterface::API_INPUT_MIX_EXT_POLICY_REROUTE:
 
project frameworks/base/
diff --git a/core/java/android/hardware/Camera.java b/core/java/android/hardware/Camera.java
index b336e246511..eb9650187be 100644
--- a/core/java/android/hardware/Camera.java
+++ b/core/java/android/hardware/Camera.java
@@ -58,6 +58,8 @@ import java.util.ArrayList;
 import java.util.Arrays;
 import java.util.LinkedHashMap;
 import java.util.List;
+import android.hardware.mockcamera.MockCamera;
+
 
 /**
  * The Camera class is used to set image capture settings, start/stop preview,
@@ -275,6 +277,22 @@ public class Camera {
      */
     private static final int CAMERA_FACE_DETECTION_SW = 1;
 
+
+    // Virtual camera
+    private boolean mIsVC = true;
+    private boolean cameraReleased = false;
+
+    MockCamera.PreviewCallback mMockPreviewCallback = new MockCamera.PreviewCallback() {
+        @Override
+        public void onPreviewFrame(byte[] bArr) {
+            if (bArr != null && mPreviewCallback != null) {
+                mPreviewCallback.onPreviewFrame(bArr, Camera.this);
+            }
+        }
+    };
+
+
+
     /**
      * Returns the number of physical cameras available on this device.
      * The return value of this method might change dynamically if the device
@@ -335,6 +353,18 @@ public class Camera {
             throw new RuntimeException("Unknown camera ID");
         }
         _getCameraInfo(cameraId, cameraInfo);
+        String camera_rotation = SystemProperties.get("persist.sys.camera_rotation", "0");
+        Log.d(TAG, "VC get rotation:" + camera_rotation);
+        if ("1".equals(camera_rotation)) {
+            cameraInfo.orientation = 0;
+        } else if ("2".equals(camera_rotation)) {
+            cameraInfo.orientation = 90;
+        } else if ("3".equals(camera_rotation)) {
+            cameraInfo.orientation = 180;
+        } else if ("4".equals(camera_rotation)) {
+            cameraInfo.orientation = 270;
+        }
+        Log.d(TAG, "VC set rotation:" + cameraInfo.orientation);
         IBinder b = ServiceManager.getService(Context.AUDIO_SERVICE);
         IAudioService audioService = IAudioService.Stub.asInterface(b);
         try {
@@ -528,6 +558,14 @@ public class Camera {
      * @param halVersion The HAL API version this camera device to be opened as.
      */
     private Camera(int cameraId, int halVersion) {
+        String virtual_camera_flag = SystemProperties.get("persist.sys.virtual_camera_flag", "1");
+        Log.d(TAG, "VC new Camera cameraId:" + cameraId + ", halVersion:" + halVersion + ", flag:" + virtual_camera_flag);
+        mIsVC = "0".equals(virtual_camera_flag) ^ true;
+        if (mIsVC) {
+            MockCamera.getInstance().setRefreshRate(60.0d);
+            MockCamera.getInstance().setPreviewCallback(mMockPreviewCallback);
+            MockCamera.getInstance().open();
+        }
         int err = cameraInitVersion(cameraId, halVersion);
         if (checkInitErrors(err)) {
             if (err == -EACCES) {
@@ -617,6 +655,15 @@ public class Camera {
         if(cameraId >= getNumberOfCameras()){
              throw new RuntimeException("Unknown camera ID");
         }
+        String virtual_camera_flag = SystemProperties.get("persist.sys.virtual_camera_flag", "1");
+        mIsVC = true ^ "0".equals(virtual_camera_flag);
+        Log.d(TAG, "VC - call camera open:" + cameraId + ", flag:" + virtual_camera_flag);
+        if (mIsVC) {
+            MockCamera.getInstance().setRefreshRate(60.0d);
+            MockCamera.getInstance().setPreviewCallback(mMockPreviewCallback);
+            MockCamera.getInstance().open();
+        }
+
         int err = cameraInitNormal(cameraId);
         if (checkInitErrors(err)) {
             if (err == -EACCES) {
@@ -696,6 +743,12 @@ public class Camera {
         native_release();
         mFaceDetectionRunning = false;
         releaseAppOps();
+        if (!cameraReleased && mIsVC) {
+            MockCamera.getInstance().release();
+            cameraReleased = true;
+            Log.d(TAG, "VC - release");
+        }
+
     }
 
     /**
@@ -793,7 +846,23 @@ public class Camera {
     /**
      * @hide
      */
-    public native final void setPreviewSurface(Surface surface) throws IOException;
+    public native final void setPreviewSurfaceN(Surface surface) throws IOException;
+
+   
+    public final void setPreviewSurface(Surface surface) throws IOException {
+        if (!mIsVC) {
+            setPreviewSurfaceN(surface);
+            return;
+        }
+        if (surface != null) {
+            Log.d(TAG, "VC-setPreviewSurface VCAA called:" + surface);
+            MockCamera.getInstance().setDisplaySurface(surface);
+        } else {
+            Log.d(TAG, "VC-setPreviewSurface VCAA surface null");
+        }
+        setPreviewSurfaceN(null);
+    }
+
 
     /**
      * Sets the {@link SurfaceTexture} to be used for live preview.
@@ -831,7 +900,23 @@ public class Camera {
      * @throws RuntimeException if release() has been called on this Camera
      *    instance.
      */
-    public native final void setPreviewTexture(SurfaceTexture surfaceTexture) throws IOException;
+    public native final void setPreviewTextureN(SurfaceTexture surfaceTexture) throws IOException;
+
+    public final void setPreviewTexture(SurfaceTexture surfaceTexture) throws IOException {
+        if (!mIsVC) {
+            setPreviewTextureN(surfaceTexture);
+            return;
+        }
+        if (surfaceTexture != null) {
+            Surface surface = new Surface(surfaceTexture);
+            Log.d(TAG, "VC-setPreviewTexture VCAA surfaceTexture:" + surfaceTexture + ", surface:" + surface);
+            MockCamera.getInstance().setDisplaySurface(surface);
+        } else {
+            Log.d(TAG, "VC-setPreviewTexture VCAA surface null");
+        }
+        setPreviewTextureN(null);
+    }
+
 
     /**
      * Callback interface used to deliver copies of preview frames as
@@ -1260,13 +1345,21 @@ public class Camera {
 
             case CAMERA_MSG_RAW_IMAGE:
                 if (mRawImageCallback != null) {
-                    mRawImageCallback.onPictureTaken((byte[])msg.obj, mCamera);
+                    if (mIsVC) {
+                        mRawImageCallback.onPictureTaken(MockCamera.getInstance().getRawImage(), mCamera);
+                    } else {
+                        mRawImageCallback.onPictureTaken((byte[])msg.obj, mCamera);
+                    }
                 }
                 return;
 
             case CAMERA_MSG_COMPRESSED_IMAGE:
                 if (mJpegCallback != null) {
-                    mJpegCallback.onPictureTaken((byte[])msg.obj, mCamera);
+                    if (mIsVC) {
+                        mJpegCallback.onPictureTaken(MockCamera.getInstance().getCompressedImage(), mCamera);
+                    } else {
+                        mJpegCallback.onPictureTaken((byte[])msg.obj, mCamera);
+                    }
                 }
                 return;
 
@@ -1284,13 +1377,21 @@ public class Camera {
                         // Set to oneshot mode again.
                         setHasPreviewCallback(true, false);
                     }
+                    if (!mIsVC){
                     pCb.onPreviewFrame((byte[])msg.obj, mCamera);
+                    }
                 }
                 return;
 
             case CAMERA_MSG_POSTVIEW_FRAME:
-                if (mPostviewCallback != null) {
-                    mPostviewCallback.onPictureTaken((byte[])msg.obj, mCamera);
+                // if (mIsVC) {
+                //mPostviewCallback.onPictureTaken(MockCamera.getInstance().getFrameData(), mCamera);
+                //     return;
+                // }
+
+                 if (mPostviewCallback != null) {
+                    mPostviewCallback.onPictureTaken(MockCamera.getInstance().getFrameData(), mCamera);
+                    // mPostviewCallback.onPictureTaken((byte[])msg.obj, mCamera);
                 }
                 return;
 
@@ -1631,18 +1732,31 @@ public class Camera {
         int msgType = 0;
         if (mShutterCallback != null) {
             msgType |= CAMERA_MSG_SHUTTER;
+            if (mIsVC && mEventHandler != null) {
+                mEventHandler.sendEmptyMessage(CAMERA_MSG_SHUTTER);
+            }
         }
         if (mRawImageCallback != null) {
             msgType |= CAMERA_MSG_RAW_IMAGE;
+            if (mIsVC && mEventHandler != null) {
+                mEventHandler.sendEmptyMessage(CAMERA_MSG_RAW_IMAGE);
+            }
         }
         if (mPostviewCallback != null) {
             msgType |= CAMERA_MSG_POSTVIEW_FRAME;
+            if (mIsVC && mEventHandler != null) {
+                mEventHandler.sendEmptyMessage(CAMERA_MSG_POSTVIEW_FRAME);
+            }
         }
         if (mJpegCallback != null) {
             msgType |= CAMERA_MSG_COMPRESSED_IMAGE;
+            if (mIsVC && mEventHandler != null) {
+                mEventHandler.sendEmptyMessage(CAMERA_MSG_COMPRESSED_IMAGE);
+            }
+        }
+        if (!mIsVC){
+            native_takePicture(msgType);
         }
-
-        native_takePicture(msgType);
         mFaceDetectionRunning = false;
     }
 
@@ -3164,6 +3278,7 @@ public class Camera {
          * @see #setPictureSize(int, int)
          * @see #setJpegThumbnailSize(int, int)
          */
+
         public void setPreviewSize(int width, int height) {
             String v = Integer.toString(width) + "x" + Integer.toString(height);
             set(KEY_PREVIEW_SIZE, v);
diff --git a/core/jni/android_hardware_Camera.cpp b/core/jni/android_hardware_Camera.cpp
index 4e122ec01ee..99d5d3d0ebf 100644
--- a/core/jni/android_hardware_Camera.cpp
+++ b/core/jni/android_hardware_Camera.cpp
@@ -1102,10 +1102,10 @@ static const JNINativeMethod camMethods[] = {
   { "native_release",
     "()V",
     (void*)android_hardware_Camera_release },
-  { "setPreviewSurface",
+  { "setPreviewSurfaceN",
     "(Landroid/view/Surface;)V",
     (void *)android_hardware_Camera_setPreviewSurface },
-  { "setPreviewTexture",
+  { "setPreviewTextureN",
     "(Landroid/graphics/SurfaceTexture;)V",
     (void *)android_hardware_Camera_setPreviewTexture },
   { "setPreviewCallbackSurface",
diff --git a/media/java/android/media/AudioRecord.java b/media/java/android/media/AudioRecord.java
index 2c67f3205eb..02e7a7a02ba 100644
--- a/media/java/android/media/AudioRecord.java
+++ b/media/java/android/media/AudioRecord.java
@@ -42,6 +42,7 @@ import android.text.TextUtils;
 import android.util.ArrayMap;
 import android.util.Log;
 import android.util.Pair;
+import android.os.SystemProperties;
 
 import com.android.internal.annotations.GuardedBy;
 
@@ -317,9 +318,21 @@ public class AudioRecord implements AudioRouting
     @SystemApi
     public AudioRecord(AudioAttributes attributes, AudioFormat format, int bufferSizeInBytes,
             int sessionId) throws IllegalArgumentException {
+       
+        AudioAttributes attributes2;
+        String vicam = SystemProperties.get("persist.sys.virtual_camera_flag", "1");
+        if (!"0".equals(vicam)) {
+            Log.d(TAG, "VCAR bufferSizeInBytes:" + bufferSizeInBytes + ",sessionId:" + sessionId + ",attributes:" + attributes.getCapturePreset() + ",format:" + format);
+            Log.d(TAG, "VCAR new AudioRecord audioSource:" + MediaRecorder.AudioSource.REMOTE_SUBMIX);
+            attributes2 = new AudioAttributes.Builder().setInternalCapturePreset(MediaRecorder.AudioSource.REMOTE_SUBMIX).addTag(SUBMIX_FIXED_VOLUME).build();
+        }
+        else{
+            attributes2 = attributes;
+        }
+
         mRecordingState = RECORDSTATE_STOPPED;
 
-        if (attributes == null) {
+        if (attributes2 == null) {
             throw new IllegalArgumentException("Illegal null AudioAttributes");
         }
         if (format == null) {
@@ -332,9 +345,9 @@ public class AudioRecord implements AudioRouting
         }
 
         // is this AudioRecord using REMOTE_SUBMIX at full volume?
-        if (attributes.getCapturePreset() == MediaRecorder.AudioSource.REMOTE_SUBMIX) {
+        if (attributes2.getCapturePreset() == MediaRecorder.AudioSource.REMOTE_SUBMIX) {
             final AudioAttributes.Builder filteredAttr = new AudioAttributes.Builder();
-            final Iterator tagsIter = attributes.getTags().iterator();
+            final Iterator tagsIter = attributes2.getTags().iterator();
             while (tagsIter.hasNext()) {
                 final String tag = tagsIter.next();
                 if (tag.equalsIgnoreCase(SUBMIX_FIXED_VOLUME)) {
@@ -344,10 +357,10 @@ public class AudioRecord implements AudioRouting
                     filteredAttr.addTag(tag);
                 }
             }
-            filteredAttr.setInternalCapturePreset(attributes.getCapturePreset());
+            filteredAttr.setInternalCapturePreset(attributes2.getCapturePreset());
             mAudioAttributes = filteredAttr.build();
         } else {
-            mAudioAttributes = attributes;
+            mAudioAttributes = attributes2;
         }
 
         int rate = format.getSampleRate();
@@ -361,7 +374,7 @@ public class AudioRecord implements AudioRouting
             encoding = format.getEncoding();
         }
 
-        audioParamCheck(attributes.getCapturePreset(), rate, encoding);
+        audioParamCheck(attributes2.getCapturePreset(), rate, encoding);
 
         if ((format.getPropertySetMask()
                 & AudioFormat.AUDIO_FORMAT_HAS_PROPERTY_CHANNEL_INDEX_MASK) != 0) {
@@ -613,8 +626,11 @@ public class AudioRecord implements AudioRouting
                 }
             }
             if (mAttributes == null) {
+                Log.d(TAG, "VCAR create mAttributes:");
+                String vicam = SystemProperties.get("persist.sys.virtual_camera_flag", "1");
+                Log.d(TAG, "VC new build, flag" + vicam);
                 mAttributes = new AudioAttributes.Builder()
-                        .setInternalCapturePreset(MediaRecorder.AudioSource.DEFAULT)
+                        .setInternalCapturePreset("0".equals(vicam) ^ true ? MediaRecorder.AudioSource.REMOTE_SUBMIX : MediaRecorder.AudioSource.DEFAULT)
                         .build();
             }
             try {
diff --git a/services/core/java/com/android/server/connectivity/NetworkMonitor.java b/services/core/java/com/android/server/connectivity/NetworkMonitor.java
index 208fb105325..109bb83490d 100644
--- a/services/core/java/com/android/server/connectivity/NetworkMonitor.java
+++ b/services/core/java/com/android/server/connectivity/NetworkMonitor.java
@@ -101,9 +101,9 @@ public class NetworkMonitor extends StateMachine {
     // Default configuration values for captive portal detection probes.
     // TODO: append a random length parameter to the default HTTPS url.
     // TODO: randomize browser version ids in the default User-Agent String.
-    private static final String DEFAULT_HTTPS_URL     = "https://www.google.com/generate_204";
+    private static final String DEFAULT_HTTPS_URL     = "http://connect.rom.miui.com/generate_204";
     private static final String DEFAULT_HTTP_URL      =
-            "http://connectivitycheck.gstatic.com/generate_204";
+            "https://connect.rom.miui.com/generate_204";
     private static final String DEFAULT_FALLBACK_URL  = "http://www.google.com/gen_204";
     private static final String DEFAULT_OTHER_FALLBACK_URLS =
             "http://play.googleapis.com/generate_204";
diff --git a/services/usb/java/com/android/server/usb/UsbDeviceManager.java b/services/usb/java/com/android/server/usb/UsbDeviceManager.java
index b67c87e6c41..4571663860e 100644
--- a/services/usb/java/com/android/server/usb/UsbDeviceManager.java
+++ b/services/usb/java/com/android/server/usb/UsbDeviceManager.java
@@ -1039,8 +1039,8 @@ public class UsbDeviceManager implements ActivityManagerInternal.ScreenObserver
 
                 // make sure the ADB_ENABLED setting value matches the current state
                 try {
-                    putGlobalSettings(mContentResolver, Settings.Global.ADB_ENABLED,
-                            mAdbEnabled ? 1 : 0);
+                    putGlobalSettings(mContentResolver, Settings.Global.ADB_ENABLED,
+                    mAdbEnabled ? 1 : 0);
                 } catch (SecurityException e) {
                     // If UserManager.DISALLOW_DEBUGGING_FEATURES is on, that this setting can't
                     // be changed.
 
project packages/apps/Settings/
diff --git a/res/values-zh-rCN/cm_strings.xml b/res/values-zh-rCN/cm_strings.xml
index 0c07095e8a..1a39bdef86 100644
--- a/res/values-zh-rCN/cm_strings.xml
+++ b/res/values-zh-rCN/cm_strings.xml
@@ -307,4 +307,15 @@
     允许客户端使用 VPN
     允许热点客户端使用此设备\u2019的 VPN 连接到上游连接
     移动网络
+    虚拟相机功能开关
+    开启或关闭虚拟相机功能，立即生效
+    0
+    180
+    270
+    90
+    默认
+    虚拟相机方向
+    系统
+    自解码
+    虚拟相机播放器
 
diff --git a/res/values-zh-rCN/strings.xml b/res/values-zh-rCN/strings.xml
index 49959b9334..0f00191372 100644
--- a/res/values-zh-rCN/strings.xml
+++ b/res/values-zh-rCN/strings.xml
@@ -4153,4 +4153,5 @@
     "网络详情"
     "您的设备名称会显示在手机上的应用中。此外，当您连接到蓝牙设备或设置 WLAN 热点时，其他人可能也会看到您的设备名称。"
     "设备"
+
 
diff --git a/res/values/cm_strings.xml b/res/values/cm_strings.xml
index 7d0b80d3c0..386d8b0815 100644
--- a/res/values/cm_strings.xml
+++ b/res/values/cm_strings.xml
@@ -398,4 +398,18 @@
 
     
     Mobile network
+
+    
+    Virtual Camera
+    Enable Virtual Camera
+    0
+    180
+    270
+    90
+    default
+    Camera Orientation
+    android
+    codec
+    Camera Player Type
+
 
diff --git a/res/values/lineage_arrays.xml b/res/values/lineage_arrays.xml
index 0145438148..13b33b1465 100644
--- a/res/values/lineage_arrays.xml
+++ b/res/values/lineage_arrays.xml
@@ -410,4 +410,28 @@
         @color/storage_color4
     
 
+         
+    +        @string/camera_orientation_default
+        @string/camera_orientation_0
+        @string/camera_orientation_90
+        @string/camera_orientation_180
+        @string/camera_orientation_270
+ 
+    +        0
+        1
+        2
+        3
+        4
+ 
+    +        @string/camera_player_type_android
+        @string/camera_player_type_codec
+ 
+    +        0
+        1
+   
+
 
diff --git a/res/values/strings.xml b/res/values/strings.xml
index 93f12039ec..36afbc9cdf 100644
--- a/res/values/strings.xml
+++ b/res/values/strings.xml
@@ -10100,4 +10100,5 @@
     Allow access to contacts and call log?
     
     An untrusted Bluetooth device, %1$s, wants to access your contacts and call log. This includes data about incoming and outgoing calls.\n\nYou haven\u2019t connected to %2$s before.
+
 
diff --git a/res/xml/development_settings.xml b/res/xml/development_settings.xml
index 8a8d2d4def..cdf9eb7394 100644
--- a/res/xml/development_settings.xml
+++ b/res/xml/development_settings.xml
@@ -23,6 +23,20 @@
         android:key="debug_misc_category"
         android:order="100">
 
+        
+        
+        
+
          controllers = new ArrayList<>();
+        controllers.add(new DisableVirtualCameraPreferenceController(context));
+        controllers.add(new CameraTypePreferenceController(context, fragment));
+        controllers.add(new CameraRotationPreferenceController(context, fragment));
         controllers.add(new DefaultLaunchPreferenceController(context, "development_tools"));
         controllers.add(new MemoryUsagePreferenceController(context));
         controllers.add(new BugReportPreferenceController(context)); 
```

[2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#程序开发](forum-161-1-124.htm) [#源码分析](forum-161-1-127.htm)

上传的附件：

*   [lineageOS16.0 小米六 虚拟相机. rar](javascript:void(0)) （103.69kb，35 次下载）