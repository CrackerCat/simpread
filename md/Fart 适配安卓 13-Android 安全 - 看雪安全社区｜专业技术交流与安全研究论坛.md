> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-290165.htm)

> Fart 适配安卓 13

本文很水，是对于不会移植 fart 的小白来写的，整个项目就只是做了一些针对高版本 fart api 的迭代，大佬请绕过，不喜勿喷。如有错误，还请指点

aosp 基于 android_13_r78，fart8.0

修改处代码中均已注释

修改点：

art_method.cc

```
extern "C" void dumpArtMethod(ArtMethod *artmethod) REQUIRES_SHARED(Locks::mutator_lock_) {
        char *dexfilepath = (char *) malloc(sizeof(char) * 1000);
        if (dexfilepath == nullptr) {
            LOG(INFO) << "ArtMethod::dumpArtMethod,methodname:" << artmethod->PrettyMethod().c_str() <<
                    "malloc 1000 byte failed";
            return;
        }
        int result = 0;
        int fcmdline = -1;
        char szCmdline[64] = {0};
        char szProcName[256] = {0};
        int procid = getpid();
        sprintf(szCmdline, "/proc/%d/cmdline", procid);
        fcmdline = open(szCmdline, O_RDONLY);
        if (fcmdline > 0) {
            result = read(fcmdline, szProcName, 256);
            if (result < 0) {
                LOG(ERROR) << "ArtMethod::dumpArtMethod,open cmdline file file error";
                return;
            }
            close(fcmdline);
        }
 
        if (szProcName[0]) {
            const DexFile *dex_file = artmethod->GetDexFile();
            const uint8_t *begin_ = dex_file->Begin(); // Start of data.
            size_t size_ = dex_file->Size(); // Length of data.
 
            memset(dexfilepath, 0, 1000);
            int size_int_ = (int) size_;
 
            memset(dexfilepath, 0, 1000);
            sprintf(dexfilepath, "/data/data/%s/%d_dexfile.dex", szProcName, size_int_);
            int dexfilefp = open(dexfilepath,O_RDONLY);
            if (dexfilefp > 0) {
                close(dexfilefp);
                dexfilefp = 0;
            } else {
                int fp = open(dexfilepath,O_CREAT | O_APPEND | O_RDWR, 0666);
                if (fp > 0) {
                    result = write(fp, (void *) begin_, size_);
                    if (result < 0) {
                    }
                    fsync(fp);
                    close(fp);
                    memset(dexfilepath, 0, 1000);
                    sprintf(dexfilepath, "/data/data/%s/%d_classlist.txt", szProcName, size_int_);
                    int classlistfile = open(dexfilepath,O_CREAT | O_APPEND | O_RDWR, 0666);
                    if (classlistfile > 0) {
                        for (size_t ii = 0; ii < dex_file->NumClassDefs(); ++ii) {
                            const dex::ClassDef &class_def = dex_file->GetClassDef(ii);
                            const char *descriptor = dex_file->GetClassDescriptor(class_def);
                            result = write(classlistfile, (void *) descriptor, strlen(descriptor));
                            if (result < 0) {
                            }
                            const char *temp = "\n";
                            result = write(classlistfile, (void *) temp, 1);
                            if (result < 0) {
                            }
                        }
                        fsync(classlistfile);
                        close(classlistfile);
                    }
                }
            }
            // const DexFile::CodeItem* code_item = artmethod->GetCodeItem();
#include "dex/standard_dex_file.h"
            LOG(ERROR) << "[dumpArtMethod->]" << "codeitem";
            //修改处
            const art::StandardDexFile::CodeItem *code_item = (art::StandardDexFile::CodeItem *) artmethod->
                    GetCodeItem();
 
            if (LIKELY(code_item != nullptr)) {
            //修改处开始
             
                CodeItemInstructionAccessor codeitemins(*dex_file, code_item);
                CodeItemDataAccessor codeitemdata(*dex_file, code_item);
                int code_item_len = 0;
                void *item = (void *) code_item;
                // if (code_item->tries_size_>0) {
                //原本的8.0直接访问tries_size，但是在源码版本13处，这个变成了私有的，但是在code_item类中
                //定义了友元，所以可以通过友元访问私有
                if (codeitemdata.TriesSize() > 0) {
                    //inline const dex::TryItem* DexFile::GetTryItems(const DexInstructionIterator& code_item_end,uint32_t offset)
                    // const uint8_t *handler_data = (const uint8_t *)(DexFile::GetTryItems(*code_item, code_item->tries_size_));
                    // uint8_t * tail = codeitem_end(&handler_data);
                    //同上，友元访问
                    const void *data_end = codeitemdata.CodeItemDataEnd();
                    code_item_len = (int) ((uint64_t) data_end - (uint64_t) item);
            //修改处结束
                } else {
                    // code_item_len = 16+code_item->insns_size_in_code_units_*2;
                    //同上，友元访问
                    code_item_len = 16 + codeitemins.InsnsSizeInCodeUnits() * 2;
                }
                memset(dexfilepath, 0, 1000);
                int size_int = (int) dex_file->Size(); // Length of data
                // uint32_t method_idx=artmethod->GetDexMethodIndexUnchecked();
                //修改处
                uint32_t method_idx = artmethod->GetDexMethodIndex();
                sprintf(dexfilepath, "/data/data/%s/%d_ins_%d.bin", szProcName, size_int, (int) gettidv1());
                int fp2 = open(dexfilepath,O_CREAT | O_APPEND | O_RDWR, 0666);
                if (fp2 > 0) {
                    lseek(fp2, 0,SEEK_END);
                    memset(dexfilepath, 0, 1000);
                    int offset = (int) ((uint64_t) item - (uint64_t) begin_);
 
                    result = write(fp2, "{name:", 6);
                    if (result < 0) {
                    }
                    std::string funcname = artmethod->PrettyMethod();
                    const char *methodname = funcname.c_str();
                    result = write(fp2, methodname, strlen(methodname));
                    if (result < 0) {
                    }
                    memset(dexfilepath, 0, 1000);
                    sprintf(dexfilepath, ",method_idx:%d,offset:%d,code_item_len:%d,ins:", method_idx, offset,
                            code_item_len);
                    int contentlength = strlen(dexfilepath);
                    result = write(fp2, (void *) dexfilepath, contentlength);
                    if (result < 0) {
                    }
 
                    long outlen = 0;
                    char *base64result = base64_encode((char *) item, (long) code_item_len, &outlen);
                    if (base64result != nullptr) {
                        result = write(fp2, base64result, outlen);
                        if (result < 0) {
                        }
                    } else {
                        const char *errorinfo = "base64encode codeitem error";
                        result = write(fp2, errorinfo, strlen(errorinfo));
                        if (result < 0) {
                        }
                    }
                    if (base64result != nullptr) {
                        free(base64result);
                        base64result = nullptr;
                    }
                    result = write(fp2, "};", 2);
                    if (result < 0) {
                    }
                    fsync(fp2);
                    close(fp2);
                }
            }
        }
 
        if (dexfilepath != nullptr) {
            free(dexfilepath);
            dexfilepath = nullptr;
        }
    }

```

[[培训]Windows 内核深度攻防：从 Hook 技术到 Rootkit 实战！](https://www.kanxue.com/book-section_list-220.htm)

[#工具脚本](forum-161-1-128.htm) [#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [fart android13.zip](javascript:void(0);) （107.30kb，16 次下载）