> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282588.htm)

> [原创]FART 移植安卓 10 整体脱壳代码详细注释

目录

*                    [整体加固脱壳部分](#整体加固脱壳部分)

### 整体加固脱壳部分

*   interpreter.cc 修改部分 (有些代码没有语义 sorry~ 主要是帮助我这种基础不好的了解 FART 作者想干啥 学习思路！！)
    
    *   ```
        namespace art {
            //add
            //声明了这个方法 类型为指向ArtMethod对象的指针
            extern "C" void beiniao_Execute(ArtMethod* artmethod);
            //add
         
        static inline JValue Execute(
            Thread* self,
            const DexFile::CodeItem* code_item,
            ShadowFrame& shadow_frame,
            JValue result_register,
            bool stay_in_interpreter = false) REQUIRES_SHARED(Locks::mutator_lock_) {
            //add
            // 检查当前方法的字符串表示是否包含 "" 子字符串。
            // 使用 strstr 函数来查找子字符串。如果找到，则返回指向子字符串的指针，否则返回空指针。
            //关于"" 类的第一次主动使用：当类第一次被主动使用时，比如创建类的实例、访问类的静态字段或调用类的静态方法。显式类加载：通过反射或者其他方式显式加载类时。
            //在 Android Runtime (ART) 中，静态初始化方法 的处理是由解释器或 JIT 编译器完成的。ART 会在类的第一次使用时检测并执行 方法。
            if (strstr(shadow_frame.GetMethod()->PrettyMethod().c_str(), "")) {
                // 如果条件为真，表示方法名中包含 ""，则执行以下代码块。
         
                // 调用函数 dumpdexfilebyExecute，将当前方法的指针传递给它。
                // 该函数的作用是导出或处理与此方法相关的 dex 文件。
                beiniao_Execute(shadow_frame.GetMethod());
            }
            //add 
        ```
        
*   art_method.cc 修改部分
    
*   ```
    //add
    #include #include #include #include #define LOG_TAG "MyTag"
    #define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
    #define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
    //add
    //add
    extern "C" void beiniao_Execute(ArtMethod * beiniao_Method) REQUIRES_SHARED(Locks::mutator_lock_) {
        //这一部分代码创建了一个名为 beiniao_path 的 std::unique_ptr 对象，并将其初始化为指向新分配的 1000 个字符的数组。beiniao_path 将自动管理这块内存，并在 beiniao_path 离开作用域时自动释放内存。
        std::unique_ptr beiniao_path(new char[1000]);
        //判断刚刚申请的内存是否申请成功
        if (beiniao_path == nullptr) {
            LOGE("ArtMethod::beiniao_Execute, methodname: %s malloc 1000 byte failed",
                 beiniao_Method->PrettyMethod().c_str());
            return;
        }
         
        int result = 0;//用于存储读取操作的结果。
        int fcmdline = -1;//用于存储文件描述符，初始化为 -1。
        char szCmdline[64] = {0};//用于存储命令行文件路径的缓冲区，大小为 64 字节，初始化为空。
        char szProcName[256] = {0};//用于存储进程名称的缓冲区，大小为 256 字节，初始化为空。
        int procid = getpid();//获取当前进程的进程ID
        //将当前进程的命令行文件路径格式化并存储在 szCmdline 缓冲区中。路径格式为 /proc//cmdline。
        sprintf(szCmdline, "/proc/%d/cmdline", procid);
        //使用 open 函数以只读模式 (O_RDONLY) 打开格式化后的命令行文件，并将文件描述符存储在 fcmdline 中。
        fcmdline = open(szCmdline, O_RDONLY);
        //判断是否打开成功
        if (fcmdline > 0) {
            //在 Linux 系统中，每个进程都有一个 /proc//cmdline 文件，包含了该进程的命令行参数。这个文件是一个以 null 字符（\0）分隔的字符串，每个参数以 null 字符结束
            //假设有一个进程的启动命令如下：
            ///usr/bin/python3 /home/user/script.py arg1 arg2
            //读取到的内容将是：
            //usr/bin/python3\0/home/user/script.py\0arg1\0arg2\0
            result = read(fcmdline, szProcName, sizeof(szProcName));
            //判断是否读取成功
            if (result < 0) {
                LOGE("ArtMethod::beiniao_Execute, read cmdline file error");
                close(fcmdline);
                return;
            }
            close(fcmdline);
        } else {
            LOGE("ArtMethod::beiniao_Execute, open cmdline file error: %s", strerror(errno));
            return;
        }
        //检查 szProcName 缓冲区的第一个字符是否非空。如果是空的，则跳过以下所有步骤。
        if (szProcName[0]) {
            const DexFile *dex_file = beiniao_Method->GetDexFile();//beiniao_begin_ 指向 Dex 文件的起始地址。
            const uint8_t *beiniao_begin_ = dex_file->Begin(); //beiniao_size_ 表示 Dex 文件的大小。
            size_t beiniao_size_ = dex_file->Size();//DexFile 类中的 Size 方法返回 Dex 文件的大小，以字节为单位。
            int size_int_ = static_cast(beiniao_size_);//通过这行代码，我们将 Dex 文件的大小从 size_t 类型转换为 int 类型，并将其存储在 size_int_ 变量中。
            //用 snprintf 函数将路径格式化并存储到 beiniao_path 中。
            snprintf(beiniao_path.get(), 1000, "/data/data/%s/beiniao", szProcName);
            //尝试创建目录，如果目录已经存在，则忽略错误。如果创建失败且错误不是目录已存在（EEXIST），则记录错误日志并返回。
            if (mkdir(beiniao_path.get(), 0777) != 0 && errno != EEXIST) {
                LOGE("ArtMethod::beiniao_Execute, mkdir error: %s", strerror(errno));
                return;
            }
            //使用 snprintf 函数将文件路径格式化并存储到 beiniao_path 中。
            snprintf(beiniao_path.get(), 1000, "/data/data/%s/beiniao/%d_dexfile_execute.dex", szProcName, size_int_);
            //使用 access 函数检查文件是否存在。如果文件存在，则记录日志并跳过文件创建步骤。
            if (access(beiniao_path.get(), F_OK) != -1) {
                LOGI("ArtMethod::beiniao_Execute, dex file already exists: %s", beiniao_path.get());
            } else {
                //使用 open 函数创建文件并获取文件描述符。如果打开成功，写入 Dex 文件数据并检查写入是否成功。如果写入成功，则同步文件内容并关闭文件描述符。
                int fp = open(beiniao_path.get(), O_CREAT | O_APPEND | O_RDWR, 0666);
                //fp: 文件描述符。如果 open 函数成功，则 fp 是一个正整数，表示打开的文件的描述符；如果 open 失败，则 fp 为 -1。
                if (fp > 0) {
                    //write 函数: 将数据写入文件。
                    //fp: 文件描述符，表示要写入的文件。
                    //beiniao_begin_: 指向要写入数据的起始位置（Dex 文件的数据）。
                    //beiniao_size_: 要写入的数据大小（Dex 文件的大小）。
                    //通过这行代码，将 Dex 文件的数据写入打开的文件中。
                    result = write(fp, beiniao_begin_, beiniao_size_);
                    //result: write 函数的返回值。如果 write 成功，则返回写入的字节数；如果 write 失败，则返回 -1。
                    if (result < 0) {
                        LOGE("ArtMethod::beiniao_Execute, write dex file error: %s", strerror(errno));
                    }
                    //fsync 函数: 将文件描述符 fp 所引用的文件的数据和元数据刷新到存储设备上，确保数据写入磁盘。
                    fsync(fp);
                    //close 函数: 关闭文件描述符 fp，释放资源。
                    close(fp);
                    //将格式化后的路径字符串存储在 beiniao_path 指向的字符数组中。
                    snprintf(beiniao_path.get(), 1000, "/data/data/%s/beiniao/%d_classlist_execute.txt", szProcName,
                             size_int_);
                    //beiniao_path.get(): 文件路径。
                    //O_CREAT: 如果文件不存在，则创建文件。
                    //O_APPEND: 以追加模式打开文件。
                    //O_RDWR: 以读写模式打开文件。
                    //返回值是文件描述符 beiniao_listfile。
                    int beiniao_listfile = open(beiniao_path.get(), O_CREAT | O_APPEND | O_RDWR, 0666);
                    //检查文件描述符是否有效，即文件是否成功打开。
                    if (beiniao_listfile > 0) {
                        //遍历 Dex 文件中的所有类定义。
                        for (size_t ii = 0; ii < dex_file->NumClassDefs(); ++ii) {
                            //获取当前类定义的引用。
                            const dex::ClassDef &class_def = dex_file->GetClassDef(ii);
                            //获取当前类定义的描述符。
                            const char *descriptor = dex_file->GetClassDescriptor(class_def);
                            //将类描述符写入文件。
                            result = write(beiniao_listfile, descriptor, strlen(descriptor));
                            //检查 write 操作是否成功。
                            if (result < 0) {
                                LOGE("ArtMethod::beiniao_Execute, write classlistfile error: %s", strerror(errno));
                            }
                            //定义一个包含换行符的字符串。
                            const char *temp = "\n";
                            //将换行符写入文件。
                            result = write(beiniao_listfile, temp, 1);
                            if (result < 0) {
                                LOGE("ArtMethod::beiniao_Execute, write classlistfile error: %s", strerror(errno));
                            }
                        }
                        //fsync 函数: 将文件描述符 beiniao_listfile 引用的文件的数据和元数据刷新到存储设备上，确保数据写入磁盘。
                        fsync(beiniao_listfile);
                        //close 函数: 关闭文件描述符 beiniao_listfile，释放资源。
                        close(beiniao_listfile);
                    } else {
                        //如果文件打开失败。记录错误日志，包含错误原因。
                        LOGE("ArtMethod::beiniao_Execute, open classlistfile error: %s", strerror(errno));
                    }
                } else {
                    LOGE("ArtMethod::beiniao_Execute, open dex file error: %s", strerror(errno));
                }
            }
        }
    }
    //add 
    ```
    

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 8 小时前 被北袅编辑 ，原因：

[#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm) [#系统相关](forum-161-1-126.htm)