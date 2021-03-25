> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/luoyesiqiu/p/fart.html)

> 本文首发于安全客
> 
> 链接：[https://www.anquanke.com/post/id/219094](https://www.anquanke.com/post/id/219094)

0x1 前言[#](#0x1-前言)
------------------

在 Android 平台上，程序员编写的 Java 代码最终将被编译成字节码在 Android 虚拟机上运行。自从 Android 进入大众的视野后，apktool,jadx 等反编译工具也层出不穷，功能也越来越强大，由 Java 编译成的字节码在这些反编译工具面前变得不堪一击，这相当于一个人裸奔在茫茫人海，身体的各个部位被众人一览无余。一种事物的出现，也会有与之对立的事物出现。有反编译工具的出现，当然也会有反反编译工具的出现，这种技术一般我们加固技术。APP 经过加固，就相当于给那个裸奔的人穿了衣服，“衣服” 在一定程度上保护了 APP，使 APP 没那么容易被反编译。当然，有加固技术的出现，也会有反加固技术的出现，即本文要分析的脱壳技术。

Android 经过多个版本的更迭，它无论在外观还是内在都有许多改变，早期的 Android 使用的是 dalvik 虚拟机，Android4.4 开始加入 ART 虚拟机，但不默认启用。从 Android5.0 开始，ART 取代 dalvik，成为默认虚拟机。由于 dalvik 和 ART 运行机制的不同，在它们内部脱壳原理也不太相同，本文分析的是 ART 下的脱壳方案：FART。它的整体思路是通过**主动调用**的方式来实现脱壳，项目地址：[https://github.com/hanbinglengyue/FART](https://github.com/hanbinglengyue/FART) 。FART 的代码是通过修改少量 Android 源码文件而成的，经过修改的 Android 源码编译成系统镜像，刷入手机，这样的手机启动后，就成为一台可以用于脱壳的脱壳机。

0x2 流程分析[#](#0x2-流程分析)
----------------------

FART 的入口在`frameworks\base\core\java\android\app\ActivityThread.java`的 performLaunchActivity 函数中，即 APP 的 Activity 启动的时候执行 fartthread

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    Log.e("ActivityThread","go into performLaunchActivity");
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    ......
    //开启fart线程
    fartthread();
    ......
}


```

fartthread 函数开启一个线程，休眠一分钟后调用 fart 函数

```
public static void fartthread() {
    new Thread(new Runnable() {

        @Override
        public void run() {
            try {
                Log.e("ActivityThread", "start sleep,wait for fartthread start......");
                Thread.sleep(1 * 60 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.e("ActivityThread", "sleep over and start fartthread");
            fart();
            Log.e("ActivityThread", "fart run over");

        }
    }).start();
}


```

fart 函数中，获取 Classloader, 反射获取一些类。反射调用`dalvik.system.DexPathList`的 dexElements 字段得到`dalvik.system.DexPathList$Element`类对象数组，Element 类存储着 dex 的路径等信息。接下来通过遍历`dexElements`，得到每一个 Element 对象中的 DexFile 对象，再获取 DexFile 对象中的 mCookie 字段值，调用 DexFile 类中的`String[] getClassNameList(Object cookie)`函数并传入获取到 mCookie，以得到 dex 文件中所有的类名。随后，遍历 dex 中的所有类名，传入`loadClassAndInvoke`函数。

```
public static void fart() {
    ClassLoader appClassloader = getClassloader();
    List<Object> dexFilesArray = new ArrayList<Object>();
    Field pathList_Field = (Field) getClassField(appClassloader, "dalvik.system.BaseDexClassLoader", "pathList");
    Object pathList_object = getFieldOjbect("dalvik.system.BaseDexClassLoader", appClassloader, "pathList");
    Object[] ElementsArray = (Object[]) getFieldOjbect("dalvik.system.DexPathList", pathList_object, "dexElements");
    Field dexFile_fileField = null;
    try {
        dexFile_fileField = (Field) getClassField(appClassloader, "dalvik.system.DexPathList$Element", "dexFile");
    } catch (Exception e) {
        e.printStackTrace();
    }
    Class DexFileClazz = null;
    try {
        DexFileClazz = appClassloader.loadClass("dalvik.system.DexFile");
    } catch (Exception e) {
        e.printStackTrace();
    }
    Method getClassNameList_method = null;
    Method defineClass_method = null;
    Method dumpDexFile_method = null;
    Method dumpMethodCode_method = null;

    for (Method field : DexFileClazz.getDeclaredMethods()) {
        if (field.getName().equals("getClassNameList")) {
            getClassNameList_method = field;
            getClassNameList_method.setAccessible(true);
        }
        if (field.getName().equals("defineClassNative")) {
            defineClass_method = field;
            defineClass_method.setAccessible(true);
        }
        if (field.getName().equals("dumpMethodCode")) {
            dumpMethodCode_method = field;
            dumpMethodCode_method.setAccessible(true);
        }
    }
    Field mCookiefield = getClassField(appClassloader, "dalvik.system.DexFile", "mCookie");
    for (int j = 0; j < ElementsArray.length; j++) {
        Object element = ElementsArray[j];
        Object dexfile = null;
        try {
            dexfile = (Object) dexFile_fileField.get(element);
        } catch (Exception e) {
            e.printStackTrace();
        }
        if (dexfile == null) {
            continue;
        }
        if (dexfile != null) {
            dexFilesArray.add(dexfile);
            Object mcookie = getClassFieldObject(appClassloader, "dalvik.system.DexFile", dexfile, "mCookie");
            if (mcookie == null) {
                continue;
            }
            String[] classnames = null;
            try {
                classnames = (String[]) getClassNameList_method.invoke(dexfile, mcookie);
            } catch (Exception e) {
                e.printStackTrace();
                continue;
            } catch (Error e) {
                e.printStackTrace();
                continue;
            }
            if (classnames != null) {
                for (String eachclassname : classnames) {
                    loadClassAndInvoke(appClassloader, eachclassname, dumpMethodCode_method);
                }
            }

        }
    }
    return;
}


```

loadClassAndInvoke 除了传入上面提到的类名，还传入 ClassLoader 对象和 dumpMethodCode 函数的 Method 对象，看上面的代码可以知道，dumpMethodCode 函数来自 DexFile, 原本的 DexFile 类没有这个函数，是 FART 加上去的。dumpMethodCode 究竟做了什么我们待会再来看，先把 loadClassAndInvoke 函数看完。loadClassAndInvoke 工作也很简单，根据传入的类名来加载类，再从加载的类获取它的所有的构造函数和函数，然后调用 dumpMethodCode，传入 Constructor 对象或者 Method 对象

```
public static void loadClassAndInvoke(ClassLoader appClassloader, String eachclassname, Method dumpMethodCode_method) {
    Log.i("ActivityThread", "go into loadClassAndInvoke->" + "classname:" + eachclassname);
    Class resultclass = null;
    try {
        resultclass = appClassloader.loadClass(eachclassname);
    } catch (Exception e) {
        e.printStackTrace();
        return;
    } catch (Error e) {
        e.printStackTrace();
        return;
    } 
    if (resultclass != null) {
        try {
            Constructor<?> cons[] = resultclass.getDeclaredConstructors();
            for (Constructor<?> constructor : cons) {
                if (dumpMethodCode_method != null) {
                    try {
                        dumpMethodCode_method.invoke(null, constructor);
                    } catch (Exception e) {
                        e.printStackTrace();
                        continue;
                    } catch (Error e) {
                        e.printStackTrace();
                        continue;
                    } 
                } else {
                    Log.e("ActivityThread", "dumpMethodCode_method is null ");
                }

            }
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        } 
        try {
            Method[] methods = resultclass.getDeclaredMethods();
            if (methods != null) {
                for (Method m : methods) {
                    if (dumpMethodCode_method != null) {
                        try {
                           dumpMethodCode_method.invoke(null, m);
                         } catch (Exception e) {
                            e.printStackTrace();
                            continue;
                        } catch (Error e) {
                            e.printStackTrace();
                            continue;
                        } 
                    } else {
                        Log.e("ActivityThread", "dumpMethodCode_method is null ");
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        } 
    }
}


```

上面提到 dumpMethodCode 函数在 DexFile 类中，DexFile 的完整路径为：`libcore\dalvik\src\main\java\dalvik\system\DexFile.java`, 它是这么定义的：

```
private static native void dumpMethodCode(Object m);


```

可见，它是一个 native 方法，它的实际代码在：`art\runtime\native\dalvik_system_DexFile.cc`，代码为：

```
static void DexFile_dumpMethodCode(JNIEnv* env, jclass,jobject method) {
ScopedFastNativeObjectAccess soa(env);
  if(method!=nullptr)
  {
		  ArtMethod* artmethod = ArtMethod::FromReflectedMethod(soa, method);
		  myfartInvoke(artmethod);
	  }	  


  return;
}


```

DexFile_dumpMethodCode 函数中，method 是 loadClassAndInvoke 函数传过来的`java.lang.reflect.Method`对象，传进来的 Java 层 Method 对象传入 FromReflectedMethod 函数得到 ArtMethod 结构指针，再将 ArtMethod 结构指针传入 myfartInvoke 函数。

myfartInvoke 实际代码在`art/runtime/art_method.cc`文件里

```
extern "C" void myfartInvoke(ArtMethod * artmethod)
 SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
	JValue *result = nullptr;
	Thread *self = nullptr;
	uint32_t temp = 6;
	uint32_t *args = &temp;
	uint32_t args_size = 6;
	artmethod->Invoke(self, args, args_size, result, "fart");
}


```

在 myfartInvoke 函数中，值得关注的是 self 被设置为空指针，并传入 ArtMethod 的 Invoke 函数。

Invoke 函数也是在`art/runtime/art_method.cc`文件里，在 Invoke 函数开头，它对 self 参数做了个判断，如果 self 为空，说明 Invoke 函数是被 FART 所调用的，反之则是系统本身的调用。self 为空的时候，调用 dumpArtMethod 函数，并立即返回

```
void ArtMethod::Invoke(Thread * self, uint32_t * args,
		       uint32_t args_size, JValue * result,
		       const char *shorty) {


	if (self == nullptr) {
		dumpArtMethod(this);
		return;
	}
    ......	
}


```

dumpArtMethod 函数这里就到了 dump dex 的代码了。

```
extern "C" void dumpArtMethod(ArtMethod * artmethod)
 SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
	char *dexfilepath = (char *) malloc(sizeof(char) * 2000);
	if (dexfilepath == nullptr) {
		LOG(INFO) <<
		    "ArtMethod::dumpArtMethodinvoked,methodname:"
		    << PrettyMethod(artmethod).
		    c_str() << "malloc 2000 byte failed";
		return;
	}
	int fcmdline = -1;
	char szCmdline[64] = { 0 };
	char szProcName[256] = { 0 };
	int procid = getpid();
	sprintf(szCmdline, "/proc/%d/cmdline", procid);
	fcmdline = open(szCmdline, O_RDONLY, 0644);
	if (fcmdline > 0) {
		read(fcmdline, szProcName, 256);
		close(fcmdline);
	}

	if (szProcName[0]) {

		const DexFile *dex_file = artmethod->GetDexFile(); 
		const char *methodname =
		    PrettyMethod(artmethod).c_str();
		const uint8_t *begin_ = dex_file->Begin(); 
		size_t size_ = dex_file->Size(); 

		memset(dexfilepath, 0, 2000);
		int size_int_ = (int) size_;

		memset(dexfilepath, 0, 2000);
		sprintf(dexfilepath, "%s", "/sdcard/fart");
		mkdir(dexfilepath, 0777);

		memset(dexfilepath, 0, 2000);
		sprintf(dexfilepath, "/sdcard/fart/%s",
			szProcName);
		mkdir(dexfilepath, 0777);

		memset(dexfilepath, 0, 2000);
		sprintf(dexfilepath,
			"/sdcard/fart/%s/%d_dexfile.dex",
			szProcName, size_int_);
		int dexfilefp = open(dexfilepath, O_RDONLY, 0666);
		if (dexfilefp > 0) {
			close(dexfilefp);
			dexfilefp = 0;

		} else {
			dexfilefp =
			    open(dexfilepath, O_CREAT | O_RDWR,
				 0666);
			if (dexfilefp > 0) {
				write(dexfilefp, (void *) begin_,
				      size_); 
				fsync(dexfilefp);
				close(dexfilefp);
			}


		}
        //下半部分开始
		const DexFile::CodeItem * code_item =
		    artmethod->GetCodeItem(); // (1)
		if (LIKELY(code_item != nullptr)) {
			int code_item_len = 0;
			uint8_t *item = (uint8_t *) code_item;
			if (code_item->tries_size_ > 0) { // (2)
				const uint8_t *handler_data = (const uint8_t *) (DexFile::GetTryItems(*code_item,code_item->tries_size_));
				uint8_t *tail = codeitem_end(&handler_data);
				code_item_len = (int)(tail - item);
			} else {
				code_item_len =
				    16 +
				    code_item->
				    insns_size_in_code_units_ * 2;
			}
			memset(dexfilepath, 0, 2000);
			int size_int = (int) dex_file->Size();	// Length of data
			uint32_t method_idx =
			    artmethod->get_method_idx();
			sprintf(dexfilepath,
				"/sdcard/fart/%s/%d_%ld.bin",
				szProcName, size_int, gettidv1());
			int fp2 =
			    open(dexfilepath,
				 O_CREAT | O_APPEND | O_RDWR,
				 0666);
			if (fp2 > 0) {
				lseek(fp2, 0, SEEK_END);
				memset(dexfilepath, 0, 2000);
				int offset = (int) (item - begin_);
				sprintf(dexfilepath,
					"{name:%s,method_idx:%d,offset:%d,code_item_len:%d,ins:",
					methodname, method_idx,
					offset, code_item_len);
				int contentlength = 0;
				while (dexfilepath[contentlength]
				       != 0)
					contentlength++;
				write(fp2, (void *) dexfilepath,
				      contentlength);
				long outlen = 0;
				char *base64result =
				    base64_encode((char *) item,
						  (long)
						  code_item_len,
						  &outlen);
				write(fp2, base64result, outlen);
				write(fp2, "};", 2);
				fsync(fp2);
				close(fp2);
				if (base64result != nullptr) {
					free(base64result);
					base64result = nullptr;
				}
			}

		}


	}

	if (dexfilepath != nullptr) {
		free(dexfilepath);
		dexfilepath = nullptr;
	}

}


```

dumpArtMethod 函数开始先通过`/proc/<pid>/cmdline`虚拟文件读取进程 pid 对应的进程名，根据得到的进程名在 sdcard 下创建目录，所以在脱壳之前要给 APP 写入外部存储的权限。之后通过 ArtMethod 的 GetDexFile 函数得到 DexFile 指针，即 ArtMethod 所在的 dex 的指针，再从 DexFile 的 Begin 函数和 Size 函数得到 dex 文件在内存中起始的地址和 dex 文件的大小，接着用 write 函数把内存中的 dex 写到文件名以`_dexfile.dex`的文件中。

但该函数还没完，dumpArtMethod 函数的下半部分，对函数的 CodeItem 进行 dump。可能有些人就有疑问了，函数的上半部分不是把 dex 给 dump 了吗，为什么还需要取函数的 CodeItem 进行 dump 呢？对于某些壳，dumpArtMethod 的上半部分已经能对 dex 进行整体 dump, 但是对于部分抽取壳，dex 即使被 dump 下来，函数体还是以 nop 填充，即空函数体，FART 还把函数的 CodeItem 给 dump 下来是让用户手动来修复这些 dump 下来的空函数。

我们来看 dumpArtMethod 函数的下半部分，这里将会涉及 dex 文件的结构，如果不了解请结合文档来看。注释`(1)`处，从 ArtMethod 中得到一个 CodeItem。注释`(2)`处，根据 CodeItem 的`tries_size_`，即 try_item 的数量来计算 CodeItem 的大小：

(1) 如果 tries_size_不为 0，说明这个 CodeItem 有 try_item，那么去把 CodeItem 的结尾地址给算出来

```
const uint8_t *handler_data = (const uint8_t *) (DexFile::GetTryItems(*code_item,code_item->tries_size_));
				uint8_t *tail = codeitem_end(&handler_data);
				code_item_len = (int)(tail - item);


```

codeitem_end 函数怎么算出 CodeItem 的结束地址呢？

GetTryItems 第二参数传入`tries_size_`，即跳过所有的 try_item，得到 encoded_catch_handler_list 的地址，然后传入 codeitem_end 函数

```
uint8_t *codeitem_end(const uint8_t ** pData) {
    uint32_t num_of_list = DecodeUnsignedLeb128(pData);
    for (; num_of_list > 0; num_of_list--) {
        int32_t num_of_handlers =
            DecodeSignedLeb128(pData);
        int num = num_of_handlers;
        if (num_of_handlers <= 0) {
            num = -num_of_handlers;
        }
        for (; num > 0; num--) {
            DecodeUnsignedLeb128(pData);
            DecodeUnsignedLeb128(pData);
        }
        if (num_of_handlers <= 0) {
            DecodeUnsignedLeb128(pData);
        }
    }
    return (uint8_t *) (*pData);
}


```

codeitem_end 函数的开头读取 encoded_catch_handler_list 结构中包含多少个 encoded_catch_handler 结构，如果不为 0，遍历所有 encoded_catch_handler 结构，读取 encoded_catch_handler 结构中有多少 encoded_type_addr_pair 结构，有的话全部跳过，即跳过了整个 encoded_catch_handler_list 结构。最后函数返回的 pData 即为 CodeItem 的结尾地址。

得到了 CodeItem 结尾地址，用 CodeItem 结尾的地址减去 CodeItem 的起始地址得到 CodeItem 的真实大小。

(2) 如果 tries_size_为 0，那么就没有 try_item，直接就能把 CodeItem 的大小计算出来：

```
code_item_len = 16 + code_item->insns_size_in_code_units_ * 2;


```

CodeItem 的大小计算出来之后，接下来可以看到，有几个变量以格式化的方式打印到 dexfilepath

```
sprintf(dexfilepath,
   "{name:%s,method_idx:%d,offset:%d,code_item_len:%d,ins:",
   methodname, 
   method_idx,
   offset, 
   code_item_len
);


```

*   name 函数的名称
*   method_idx 来源 FART 新增的函数：`uint32_t get_method_idx(){ return dex_method_index_; }`, 函数返回 dex_method_index_，dex_method_index_是函数在`method_ids`中的索引
*   offset 是该函数的 CodeItem 相对于 dex 文件开始的偏移
*   code_item_len CodeItem 的长度

数据组装好之后，写入到以`.bin`为后缀的文件中：

```
write(fp2, (void *) dexfilepath,
        contentlength);
long outlen = 0;
char *base64result =
    base64_encode((char *) item,
            (long)
            code_item_len,
            &outlen);
write(fp2, base64result, outlen);
write(fp2, "};", 2);


```

对于上面的 dexfilepath，它们是明文字符，直接写入即可。而对于 CodeItem 中的 bytecode 这种非明文字符，直接写入不太好看，所以 FART 选择对它们进行 base64 编码后再写入。

分析到这里好像已经结束了，从主动调用，到 dex 整体 dump，再到函数 CodeItem 的 dump，都已经分析了。但是 FART 中确实还有一部分逻辑是没有分析的。如果你使用过 FART 来脱过壳，会发现它 dump 下来的 dex 中还有以`_execute.dex`结尾的 dex 文件。这种 dex 是怎么生成的呢？

这一部分的代码也是在`art\runtime\art_method.cc`文件中

```
	extern "C" void dumpDexFileByExecute(ArtMethod * artmethod)
	 SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
		char *dexfilepath = (char *) malloc(sizeof(char) * 2000);
		if (dexfilepath == nullptr) {
			LOG(INFO) <<
			    "ArtMethod::dumpDexFileByExecute,methodname:"
			    << PrettyMethod(artmethod).
			    c_str() << "malloc 2000 byte failed";
			return;
		}
		int fcmdline = -1;
		char szCmdline[64] = { 0 };
		char szProcName[256] = { 0 };
		int procid = getpid();
		sprintf(szCmdline, "/proc/%d/cmdline", procid);
		fcmdline = open(szCmdline, O_RDONLY, 0644);
		if (fcmdline > 0) {
			read(fcmdline, szProcName, 256);
			close(fcmdline);
		}

		if (szProcName[0]) {

			const DexFile *dex_file = artmethod->GetDexFile();
			const uint8_t *begin_ = dex_file->Begin();	// Start of data.
			size_t size_ = dex_file->Size();	// Length of data.

			memset(dexfilepath, 0, 2000);
			int size_int_ = (int) size_;

			memset(dexfilepath, 0, 2000);
			sprintf(dexfilepath, "%s", "/sdcard/fart");
			mkdir(dexfilepath, 0777);

			memset(dexfilepath, 0, 2000);
			sprintf(dexfilepath, "/sdcard/fart/%s",
				szProcName);
			mkdir(dexfilepath, 0777);

			memset(dexfilepath, 0, 2000);
			sprintf(dexfilepath,
				"/sdcard/fart/%s/%d_dexfile_execute.dex",
				szProcName, size_int_);
			int dexfilefp = open(dexfilepath, O_RDONLY, 0666);
			if (dexfilefp > 0) {
				close(dexfilefp);
				dexfilefp = 0;

			} else {
				dexfilefp =
				    open(dexfilepath, O_CREAT | O_RDWR,
					 0666);
				if (dexfilefp > 0) {
					write(dexfilefp, (void *) begin_,
					      size_);
					fsync(dexfilefp);
					close(dexfilefp);
				}


			}
		}

		if (dexfilepath != nullptr) {
			free(dexfilepath);
			dexfilepath = nullptr;
		}

	}


```

可以看到，dumpDexFileByExecute 函数有点像 dumpArtMethod 函数的上半部分，即对 dex 文件的整体 dump。那么，dumpDexFileByExecute 在哪里被调用呢？

通过搜索，在`art\runtime\interpreter\interpreter.cc`文件的开始，看到了 FART 在 art 命名空间下定义了一个 dumpDexFileByExecute 函数

```
namespace art {
extern "C" void dumpDexFileByExecute(ArtMethod* artmethod);
namespace interpreter {
        ......
    }
}


```

同时在文件其中找到了对 dumpDexFileByExecute 函数的调用：

```
static inline JValue Execute(Thread* self, const DexFile::CodeItem* code_item,
                             ShadowFrame& shadow_frame, JValue result_register) { 
  if(strstr(PrettyMethod(shadow_frame.GetMethod()).c_str(),"<clinit>")!=nullptr)
  {
	  dumpDexFileByExecute(shadow_frame.GetMethod());
  }
  ......
}


```

在 Execute 函数中，通过判断函数名称中是否为`<clinit>`决定要不要调用 dumpDexFileByExecute，即判断传入的是否为静态代码块，对于加了壳的 App 来说静态代码块是肯定存在的。如果 Execute 传入的是静态代码块则调用 dumpDexFileByExecute 函数，并传入一个 ArtMethod 指针。

dumpDexFileByExecute 中对 dex 进行了整体 dump，可以把它看作是 dumpArtMethod 方式的互补，有时 dumpArtMethod 中得不到想得到的 dex, 用 dumpDexFileByExecute 或许能得到惊喜。

0x3 结语[#](#0x3-结语)
------------------

非常感谢 FART 作者能够开源 FART，这使得人们对抗 ART 环境下 App 壳得到了良好的思路。FART 脱壳机理论上来讲能脱大多数壳，但是仍有例外，需要自行摸索。

0x4 参考[#](#0x4-参考)
------------------

*   [https://bbs.pediy.com/thread-252630.htm](https://bbs.pediy.com/thread-252630.htm)
*   [https://source.android.google.cn/devices/tech/dalvik/dex-format](https://source.android.google.cn/devices/tech/dalvik/dex-format)

[![](http://images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)](//images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)

**关注微信公众号：luoyesiqiu，浏览更多内容**