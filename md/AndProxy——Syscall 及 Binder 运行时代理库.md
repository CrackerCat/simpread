> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2100559-1-1.html)

> [md]# Native 层 binder 与 syscall 代理技术灵感来源：[【Android】深入 Binder 底层拦截](https://www.52pojie.cn/forum.php?mod=view......

 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 孤木落

Native 层 binder 与 syscall 代理技术
------------------------------

灵感来源：[【Android】深入 Binder 底层拦截](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1866468)

Github 项目名：AndProxyDemo

由于本人是个菜鸟（自学 android 野路子这一块），并不是很了解 binder 通信的实际原理与调用链，所以 AndProxy 其实还是有很多问题。而且学业繁忙，检查问题与修复 bug 能力有限，期待更多大佬贡献代码。

### 代理技术原理

*   Syscall 代理：源于我对老文章[某去签工具分析](https://ggggmllll.github.io/2025/12/20/%E6%9F%90%E5%8E%BB%E7%AD%BE%E5%B7%A5%E5%85%B7%E5%88%86%E6%9E%90/)（这里有一个兼容性问题，`seccomp_notify`机制需要内核版本大于 5.10，所以一般要出场 Android 12 以上的机型才可用）
*   ioctl hook：就是简单的 GotHook。但是有两个比较重要的细节（坑）
    1.  由于 Android 的命名空间机制，`dl_iterate_phdr`无法找到 libbinder.so，只能手动扫描 / proc/self/maps
    2.  Got 项没有写权限，必须先`mprotect`赋予`w`可写再修改，最后把权限改回去
*   binder 数据解析与修改：参考[【Android】深入 Binder 底层拦截](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1866468)

### Syscall 代理技术的一些细节

其实这个框架最早是作为我写的一个拦截皇室战争的`exit_group`等退出函数的 frida 模块被开发出的。为什么不使用已经成熟的 seccomp+sigaction 机制？因为皇室战争的加固提供方 Promon 已经对此通过主动注册信号处理函数作了防范，但是 seccomp_notify 处于兼容性原因，暂时没有进行防范。

在拦截退出时，我们有一个基本需求：打印推出前的函数调用栈，快速判断触发闪退位置。

但是，`seccomp_notify`机制本身不支持获取 syscall 被调用时的寄存器上下文，所以我们必须利用额外的机制，自然联想到信号量。基本流程是，若要获取寄存器，则在 supervisor_loop 启动前注册一个信号处理函数，syscall 被调用时会触发 supervisior，此时 supervisor 会接收到当前寄存器，再通过 shared memory 传回 supervisior。优点是可以利用 SIG_USER2 这种很少被注册处理函数，但是可以触发处理函数的信号，这样防御成本就大大增加了。

然后就是第一个坑，通过查阅文档，syscall 调用后，进程处于阻塞状态，此时如果发送了信号，就会打断阻塞，syscall 也不会被调用。为此，我们必须将 sig 的 flag 加上 restart，这样就可以再次调用 syscall 进入 supervisor。我们将在第一次进入 supervisor 进入时获取寄存器上下文，在第二次进入时调用回调函数进行真正的处理。

```
static void supervisor_loop(int lfd, int cmd_fd, int use_signal) {
    LOGD("supervisor_loop started, lfd=%d", lfd);
    g_notify_fd = lfd;
    struct seccomp_notif_sizes sizes{};
    if (seccomp(SECCOMP_GET_NOTIF_SIZES, 0, &sizes) != 0) {
        LOGE("GET_NOTIF_SIZES failed\n");
        _exit(-1);
    }

    auto *req = static_cast<struct seccomp_notif *>(malloc(sizes.seccomp_notif));
    auto *resp = static_cast<struct seccomp_notif_resp *>(malloc(sizes.seccomp_notif_resp));
    if (!req || !resp) {
        LOGE("malloc failed\n");
        _exit(-1);
    }

    struct pollfd fds[2];
    fds[0].fd = lfd;
    fds[0].events = POLLIN;
    fds[1].fd = cmd_fd;
    fds[1].events = POLLIN;

    int is_first_enter = 1;

    while (true) {
        int ret = poll(fds, 2, -1);
        if (ret < 0) {
            if (errno == EINTR) continue;
            break;
        }

        // 处理 seccomp 通知
        if (fds[0].revents & POLLIN) {
            memset(req, 0, sizes.seccomp_notif);
            if (ioctl(lfd, SECCOMP_IOCTL_NOTIF_RECV, req) < 0) {
                if (errno == EINTR) continue;
                LOGE("NOTIF_RECV failed: %d\n", errno);
                break;
            }
            LOGD("supervisor: received notification id=%llu, nr=%d, pid=%d",
                 req->id, req->data.nr, req->pid);

            hook_regs_t regs = {0};
            if (use_signal && is_first_enter) {
                if (tgkill(g_parent_pid, static_cast<pid_t>(req->pid), SIGUSR2) == 0) {
                    LOGD("supervisor: tgkill sent to pid=%d", req->pid);
                    int timeout = 100;
                    while (timeout-- > 0 && g_shared->ready == 0) {
                        usleep(1000);
                    }
                    if (g_shared->ready) {
                        regs = g_shared->regs;
                        g_shared->ready = 0;
                        LOGD("supervisor: pc %llx from reg_pipe", regs.pc);
                    }
                    is_first_enter = 0;
                    continue;
                } else {
                    LOGE("tgkill failed: %d\n", errno);
                }
            } else if (use_signal) {
                is_first_enter = 1;
            }

            hook_request_t hook_req;
            hook_req.id = req->id;
            hook_req.syscall_nr = req->data.nr;
            hook_req.pid = static_cast<pid_t>(req->pid);
            memcpy(hook_req.args, req->data.args, sizeof(req->data.args));
            hook_req.regs = regs;

            hook_response_t hook_resp;
            memset(&hook_resp, 0, sizeof(hook_resp));
            hook_resp.id = req->id;
            hook_resp.action = HOOK_ACTION_ALLOW;

            // 执行回调
            LOGD("supervisor: before callbacks");
            // pthread_mutex_lock(&g_hook_lock);
            hook_node_t *node = g_hook_list;
            while (node) {
                if (node->nr == hook_req.syscall_nr) {
                    node->callback(&hook_req, &hook_resp, node->userdata);
                }
                node = node->next;
            }
            // pthread_mutex_unlock(&g_hook_lock);
            LOGD("supervisor: after callbacks");

            memset(resp, 0, sizes.seccomp_notif_resp);
            resp->id = req->id;
            if (hook_resp.action == HOOK_ACTION_ALLOW) {
                resp->flags = SECCOMP_USER_NOTIF_FLAG_CONTINUE;
                resp->error = 0;
                resp->val = 0;
            } else if (hook_resp.action == HOOK_ACTION_DENY) {
                resp->flags = 0;
                resp->error = hook_resp.error;
                resp->val = 0;
            } else {
                resp->flags = 0;
                resp->error = 0;
                resp->val = hook_resp.val;
            }

            if (ioctl(lfd, SECCOMP_IOCTL_NOTIF_ID_VALID, &req->id) == 0) {
                if (ioctl(lfd, SECCOMP_IOCTL_NOTIF_SEND, resp) < 0) {
                    LOGE("SEND error: %d\n", errno);
                }
            }
        }

        // 处理父进程命令
        if (fds[1].revents & POLLIN) {
            cmd_t cmd;
            ssize_t n = read(cmd_fd, &cmd, sizeof(cmd));
            if (n == sizeof(cmd)) {
                if (cmd.type == CMD_REGISTER) {
                    seccomp_hook_register(cmd.nr, (hook_callback_t)cmd.callback, (void*)cmd.userdata);
                } else if (cmd.type == CMD_UNREGISTER) {
                    seccomp_hook_unregister(cmd.nr, (hook_callback_t)cmd.callback, (void*)cmd.userdata);
                }
            } else if (n == 0) {
                break;
            } else {
                LOGE("read cmd failed: %zd\n", n);
            }
        }
    }

    free(req);
    free(resp);
    close(lfd);
    close(cmd_fd);
    _exit(0);
}

```

第二个坑是，supervisor 和主进程不是同一个进程，所以注册 callback 时可能会有 bug。我主要利用的是 fork 不会改变模块布局，所以如果模块在 supervisor 启动前就已经被加载，那么他的函数指针在 supervisor 和被监控进程中是等价的，直接传递即可。

```
int seccomp_hook_register_remote(const int *syscall_list,
                                 const struct sock_fprog *custom_filter, hook_callback_t callback, void *userdata) {
    if (!syscall_list || *syscall_list == -1) return -1;
    struct sock_fprog prog{};
    if (custom_filter) {
        prog = *custom_filter;
    } else {
        prog = build_filter(syscall_list);
    }
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog) < 0) {
        LOGE("failed to add filter");
    }
    for (const int* nr = syscall_list; *nr != -1; nr++) {
        cmd_t cmd;
        cmd.type = CMD_REGISTER;
        cmd.nr = *nr;
        cmd.callback = (uintptr_t)callback;
        cmd.userdata = (uintptr_t)userdata;
        ssize_t w = write(g_cmd_pipe_wr, &cmd, sizeof(cmd));
        if (w != sizeof(cmd))
            return -1;
    }
    return 0;
}

int seccomp_hook_unregister_remote(const int *syscall_list, hook_callback_t callback, void *userdata) {
    for (const int* nr = syscall_list; *nr != -1; nr++) {
        cmd_t cmd;
        cmd.type = CMD_UNREGISTER;
        cmd.nr = *nr;
        cmd.callback = (uintptr_t)callback;
        cmd.userdata = (uintptr_t)userdata;
        ssize_t w = write(g_cmd_pipe_wr, &cmd, sizeof(cmd));
        if (w != sizeof(cmd))
            return -1;
    }
    return 0;
}

```

最后是如何在被监控进程调用可能被 seccomp 限制的回调？这个利用了 seccomp 的限制以线程为单位的特性，执行 java callback 的线程先于 seccomp 的 bpf 规则注册就不会被限制，所以会在 JNI_OnLoad 中先启动线程，init 函数在此之后调用。（从这个角度看 SvcInterceptor 的 addFile 方法其实没有什么用，可能会在后续版本移除）

```
extern "C" JNIEXPORT jint JNICALL Java_com_gumuluo_proxy_SvcInterceptor_init
        (JNIEnv *env, jclass clazz, jintArray syscallList) {
    if (syscallList == nullptr) return -1;
    if (g_handler_running > 0) return 0;

    // 1. 解析系统调用号数组（与之前相同）
    jsize len = env->GetArrayLength(syscallList);
    jint *elems = env->GetIntArrayElements(syscallList, nullptr);
    if (!elems) return -1;

    int hasTerm = 0;
    for (int i = 0; i < len; i++) {
        if (elems[i] == -1) { hasTerm = 1; break; }
    }

    int *c_list;
    int list_len;
    if (hasTerm) {
        c_list = (int*)malloc(len * sizeof(int));
        memcpy(c_list, elems, len * sizeof(int));
        list_len = len;
    } else {
        c_list = (int*)malloc((len + 1) * sizeof(int));
        memcpy(c_list, elems, len * sizeof(int));
        c_list[len] = -1;
        list_len = len + 1;
    }
    env->ReleaseIntArrayElements(syscallList, elems, JNI_ABORT);

    int num_entries = list_len - 1;
    auto *entries = (hook_entry_t*)malloc(num_entries * sizeof(hook_entry_t));
    if (!entries) { free(c_list); return -1; }
    for (int i = 0; i < num_entries; i++) {
        entries[i].nr = c_list[i];
        entries[i].callback = global_java_callback;
        entries[i].userdata = nullptr;
    }

    // 2. 创建 socketpair（父子进程通信）
    int sv[2];
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, sv) < 0) {
        LOGE("socketpair failed");
        free(c_list); free(entries);
        return -1;
    }
    g_sock_fd = sv[0];        // 父进程端（用于 handler 线程）
    g_child_sock_fd = sv[1];  // 子进程端（fork 后子进程使用）

    // 3. 启动处理线程（父线程）
    pthread_create(&g_handler_thread, nullptr, handler_thread_func, (void*)(intptr_t)g_sock_fd);
    g_handler_running = 1;

    // 4. 调用库初始化（内部会 fork 子进程）
    int ret = seccomp_hook_init(c_list, nullptr, entries, num_entries, 1);
    free(c_list);
    free(entries);

    if (ret != 0) {
        // 初始化失败，关闭 socket，线程将因 recv 错误而退出
        close(g_sock_fd);
        close(g_child_sock_fd);
        return ret;
    }

    // 5. 父进程关闭子进程端（子进程已有副本）
    close(g_child_sock_fd);
    g_child_sock_fd = -1;

    return 0;
}

```

> 具体实现参考：seccomp_hook.cpp, SvcInterceptor.cpp

### binder 代理的一些细节

感谢 @iofomo（数字锋芒）大佬的的无私分享，主要技术细节都在他的 blog 中，这里只分享几个小问题的解决方案。

首先是服务名解析（Stub 中的 DESCRIPTOR 字段），按照大佬分享的方法没有解析出来。但是注意到 [strlen][utf16_str] 的固定结构，采用了模糊搜索方法，成功解析服务名（缺点是很容易误匹配，所以我加了很多限制）。

```
std::string get_server_name(const binder_transaction_data* txn) {
    if (!txn || !txn->data.ptr.buffer || txn->data_size < 16) {
        return "";
    }

    const uint8_t* base = reinterpret_cast<const uint8_t*>(
            static_cast<uintptr_t>(txn->data.ptr.buffer));
    size_t size = txn->data_size;

    // 辅助函数：从给定位置提取 UTF-16 字符串
    auto extract_utf16 = [&](size_t offset, int32_t len) -> std::string {
        if (offset + 4 + len * 2 > size) return "";
        const uint16_t* name16 = reinterpret_cast<const uint16_t*>(base + offset + 4);
        std::string result;
        result.reserve(len);
        for (int32_t i = 0; i < len; ++i) {
            uint16_t ch = name16[i];
            if (ch < 0x80) {
                result.push_back(static_cast<char>(ch));
            } else if (ch < 0x800) {
                result.push_back(static_cast<char>(0xC0 | (ch >> 6)));
                result.push_back(static_cast<char>(0x80 | (ch & 0x3F)));
            } else {
                result.push_back(static_cast<char>(0xE0 | (ch >> 12)));
                result.push_back(static_cast<char>(0x80 | ((ch >> 6) & 0x3F)));
                result.push_back(static_cast<char>(0x80 | (ch & 0x3F)));
            }
        }
        return result;
    };

    // 服务名合法性检查：只允许字母、数字、点、下划线、美元符号
    auto is_valid_name = [](const std::string& s) -> bool {
        if (s.empty()) return false;
        for (char c : s) {
            if (!((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') ||
                  (c >= '0' && c <= '9') || c == '.' || c == '_' || c == '$')) {
                return false;
            }
        }
        return true;
    };

    // 首先寻找 "TSYS" 标志（flat_binder_object.hdr.type == BINDER_TYPE_BINDER）
    const uint32_t TSYS_MAGIC = 0x54535953;  // "TSYS"
    for (size_t offset = 8; offset + 8 <= size; ++offset) {
        if (*(uint32_t*)(base + offset) == TSYS_MAGIC) {
            size_t len_pos = offset + 4;
            if (len_pos + 4 > size) continue;
            int32_t nameLen = *(int32_t*)(base + len_pos);
            if (nameLen > 3 && nameLen <= 256) {
                std::string candidate = extract_utf16(len_pos, nameLen);
                if (!candidate.empty() && is_valid_name(candidate) &&
                    candidate.find('.') != std::string::npos) {
                    return candidate;
                }
            }
        }
    }

    // 回退到模糊搜索
    for (size_t offset = 0; offset + 6 <= size; ++offset) {
        const int32_t* len_ptr = reinterpret_cast<const int32_t*>(base + offset);
        int32_t nameLen = *len_ptr;
        if (nameLen <= 3 || nameLen > 256) continue;
        size_t str_bytes = static_cast<size_t>(nameLen) * 2;
        if (offset + 4 + str_bytes > size) continue;

        const uint16_t* name16 = reinterpret_cast<const uint16_t*>(base + offset + 4);
        bool valid = true;
        for (int32_t i = 0; i < nameLen; ++i) {
            uint16_t ch = name16[i];
            if ((ch & 0xFF00) == 0) {
                uint8_t lo = ch & 0xFF;
                if (lo < 0x20 || lo > 0x7E) {
                    valid = false;
                    break;
                }
            }
        }
        if (!valid) continue;

        std::string result = extract_utf16(offset, nameLen);
        if (is_valid_name(result)) {
            return result;
        }
    }

    return "";
}

```

然后是为什么选择 Got Hook？因为我不想在代码段留脏页，不想改 rx 内存的权限为可写（bushi）。其实是因为 Got Hook 比较简单，而且 binder 通信位置相对集中在 libbinder.so，所以选择对 libbinder.so 进行 Got Hook。

```
// 通过 /proc/self/maps 获取库基址
static uintptr_t get_library_base(const char *libname) {
    char line[512];
    FILE *fp = fopen("/proc/self/maps", "r");
    if (!fp) {
        LOGE("Failed to open /proc/self/maps: %s", strerror(errno));
        return 0;
    }

    uintptr_t base = 0;
    while (fgets(line, sizeof(line), fp)) {
        char *path = strchr(line, '/');
        if (path) {
            if (strstr(path, libname) != NULL) {
                char *dash = strchr(line, '-');
                if (dash) {
                    *dash = '\0';
                    base = strtoull(line, NULL, 16);
                    LOGD("Found library %s at base 0x%lx", libname, base);
                    break;
                } else {
                    LOGE("Invalid map line (no dash): %s", line);
                }
            }
        }
    }
    fclose(fp);
    if (base == 0) {
        LOGE("Library %s not found in /proc/self/maps", libname);
    }
    return base;
}

// 从动态段中获取指定类型的信息（返回偏移量，需加 base）
static uintptr_t get_dynamic_info_offset(const Elf64_Dyn *dyn, int64_t tag) {
    for (; dyn->d_tag != DT_NULL; ++dyn) {
        if (dyn->d_tag == tag) {
            return dyn->d_un.d_ptr;
        }
    }
    return 0;
}

// 根据函数名在指定库中查找 GOT 条目地址
static uintptr_t find_got_entry(const char *libname, const char *funcname) {
    uintptr_t base = get_library_base(libname);
    if (base == 0) {
        LOGE("Failed to get base address for %s", libname);
        return 0;
    }

    // 读取 ELF 头
    Elf64_Ehdr *ehdr = (Elf64_Ehdr*)base;
    if (ehdr->e_ident[EI_MAG0] != ELFMAG0 ||
        ehdr->e_ident[EI_MAG1] != ELFMAG1 ||
        ehdr->e_ident[EI_MAG2] != ELFMAG2 ||
        ehdr->e_ident[EI_MAG3] != ELFMAG3) {
        LOGE("Invalid ELF header at base 0x%lx", base);
        return 0;
    }
    LOGD("Valid ELF header at base 0x%lx", base);

    // 找到 PT_DYNAMIC 程序头
    Elf64_Phdr *phdr = (Elf64_Phdr*)(base + ehdr->e_phoff);
    Elf64_Dyn *dyn = NULL;
    for (int i = 0; i < ehdr->e_phnum; ++i) {
        if (phdr[i].p_type == PT_DYNAMIC) {
            dyn = (Elf64_Dyn*)(base + phdr[i].p_vaddr);
            LOGD("Found PT_DYNAMIC at offset 0x%lx, vaddr 0x%lx",
                 phdr[i].p_offset, phdr[i].p_vaddr);
            break;
        }
    }
    if (!dyn) {
        LOGE("No PT_DYNAMIC segment found");
        return 0;
    }

    // 获取动态段中的关键信息（偏移量）
    uintptr_t symtab_off = get_dynamic_info_offset(dyn, DT_SYMTAB);
    uintptr_t strtab_off = get_dynamic_info_offset(dyn, DT_STRTAB);
    uintptr_t relplt_off = get_dynamic_info_offset(dyn, DT_JMPREL);
    size_t relplt_size = get_dynamic_info_offset(dyn, DT_PLTRELSZ);
    uintptr_t pltgot_off = get_dynamic_info_offset(dyn, DT_PLTGOT);

    if (!symtab_off) {
        LOGE("DT_SYMTAB not found");
        return 0;
    }
    if (!strtab_off) {
        LOGE("DT_STRTAB not found");
        return 0;
    }
    if (!relplt_off) {
        LOGE("DT_JMPREL not found");
        return 0;
    }
    if (!relplt_size) {
        LOGE("DT_PLTRELSZ is zero");
        return 0;
    }
    if (!pltgot_off) {
        LOGE("DT_PLTGOT not found");
        return 0;
    }

    // 计算实际内存地址（基址 + 偏移）
    Elf64_Sym *symtab = (Elf64_Sym*)(base + symtab_off);
    const char *strtab = (const char*)(base + strtab_off);
    Elf64_Rela *relplt = (Elf64_Rela*)(base + relplt_off);
    uintptr_t pltgot = base + pltgot_off;

    LOGD("symtab=0x%lx (offset 0x%lx), strtab=0x%lx, relplt=0x%lx, relplt_size=%zu, pltgot=0x%lx",
         (uintptr_t)symtab, symtab_off, (uintptr_t)strtab, (uintptr_t)relplt, relplt_size, pltgot);

    // 遍历 .rel.plt 重定位表
    size_t num_rel = relplt_size / sizeof(Elf64_Rela);
    LOGD("Number of relocations in .rel.plt: %zu", num_rel);
    for (size_t i = 0; i < num_rel; ++i) {
        uint32_t sym_idx = ELF64_R_SYM(relplt[i].r_info);
        const char *sym_name = strtab + symtab[sym_idx].st_name;
        if (strcmp(sym_name, funcname) == 0) {
            uintptr_t got_entry = base + relplt[i].r_offset;  // r_offset 也是偏移，需加基址
            LOGD("Found GOT entry for %s at 0x%lx (r_offset=0x%lx)",
                 funcname, got_entry, relplt[i].r_offset);
            return got_entry;
        }
    }

    LOGE("Function %s not found in .rel.plt", funcname);
    return 0;
}

static void got_hook(uintptr_t got_addr, void *new_func, void **old_func) {
    void **got_ptr = (void**)got_addr;
    void *orig = *got_ptr;

    long page_size = sysconf(_SC_PAGESIZE);
    if (page_size <= 0) {
        LOGE("sysconf(_SC_PAGESIZE) failed: %s", strerror(errno));
        return;
    }

    uintptr_t page_start = got_addr & ~(page_size - 1);
    LOGD("Got address 0x%lx, page_start 0x%lx, page_size %ld",
         got_addr, page_start, page_size);

    if (mprotect((void*)page_start, page_size, PROT_READ | PROT_WRITE) == -1) {
        LOGE("mprotect (write) failed: %s", strerror(errno));
        return;
    }

    // 执行 GOT 替换
    *got_ptr = new_func;
    LOGD("GOT entry at 0x%lx changed from %p to %p", got_addr, orig, new_func);

    if (mprotect((void*)page_start, page_size, PROT_READ) == -1) {
        LOGE("mprotect (read-only) failed: %s", strerror(errno));
        // 即使恢复失败，hook 已生效，只打印警告
    }

    if (old_func) *old_func = orig;
}

void BinderHook::init(JavaVM* vm) {
    uintptr_t got_entry = find_got_entry("libbinder.so", "ioctl");
    if (!got_entry) {
        LOGE("Failed to find GOT entry for ioctl");
        return;
    }

    got_hook(got_entry, (void*) ioctl_proxy, NULL);
    jvm_ = vm;
    JNIEnv* env;
    if (jvm_->GetEnv((void**)&env, JNI_VERSION_1_6) != JNI_OK) {
        jvm_->AttachCurrentThread(&env, nullptr);
    }

    jclass dispatcherClass = env->FindClass("com/gumuluo/proxy/binder/BinderDispatcher");
    dispatcherClass_ = (jclass)env->NewGlobalRef(dispatcherClass);
    dispatchBeforeMid_ = env->GetStaticMethodID(dispatcherClass_, "dispatchBefore",
                                                "(Ljava/lang/String;Ljava/lang/String;Landroid/os/Parcel;Landroid/os/Parcel;)Z");
    dispatchAfterMid_ = env->GetStaticMethodID(dispatcherClass_, "dispatchAfter",
                                               "(Ljava/lang/String;Ljava/lang/String;Landroid/os/Parcel;Landroid/os/Parcel;)Z");

    env->DeleteLocalRef(dispatcherClass);
}

```

最后是解包 Parcel 对象，这个放在 java 层做比较简单，具体示例在`app`模块中有演示，注意一定要清除缓存

```
private fun enableFraud() {
        // 先注销已有的拦截器（如果有），避免重复
        BinderDispatcher.unregisterAfter("android.content.pm.IPackageManager", "getApplicationInfo")
        CacheHandling.clearCaches()

        // 注册新的 after 回调
        BinderDispatcher.registerAfter(
            "android.content.pm.IPackageManager",
            "getApplicationInfo",
            BinderInterceptor { data, outReply ->
                Log.d("Bypass", "========== Interceptor triggered ==========")
                Log.d("Bypass", "dataSize=${data.dataSize()}, dataAvail=${data.dataAvail()}")

                try {
                    // 1. 读取异常（安全跳过所有异常相关信息）
                    data.readException()
                } catch (e: Exception) {
                    // 如果服务返回了异常，记录日志并继续处理（但通常不会）
                    Log.w("Bypass", "Service threw exception: $e")
                    // 注意：这里不能直接返回 false，因为我们需要继续处理数据（可能有部分数据）
                }

                // 2. 读取 ApplicationInfo 对象（可能为 null）
                val originalInfo = data.readTypedObject(ApplicationInfo.CREATOR)
                if (originalInfo == null) {
                    Log.w("Bypass", "No ApplicationInfo returned")
                    return@BinderInterceptor false
                }

                // 3. 修改 flags
                val originalFlags = originalInfo.flags
                originalInfo.flags = originalFlags and ApplicationInfo.FLAG_DEBUGGABLE.inv()
                Log.d("Bypass", "Modified flags: 0x${Integer.toHexString(originalFlags)} -> 0x${Integer.toHexString(originalInfo.flags)}")

                // 4. 构造新的 outReply，模仿服务端的构造方式
                outReply.setDataPosition(0)
                outReply.writeNoException()                 // 写入异常码及可能的 StrictMode 信息
                outReply.writeTypedObject(originalInfo, 0)  // 写入对象标志 + 对象数据

                Log.d("Bypass", "Modified data size: ${outReply.dataSize()}")
                true
            }
        )

        // 重新获取状态（此时拦截器已生效）
        val isDebuggable = getDebuggableFlag()
        resultText = if (isDebuggable) "欺诈后状态: DEBUGGABLE = true" else "欺诈后状态: DEBUGGABLE = false"
    }

```

### 技术展望

我开发 AndProxy 的最初目的是做 android 运行时监控，但是 AndProxy 还可以用于做隔离沙盒，PMS Hook，IO 重定向等等。目前我计划将其用于开发 APP 多开和监控沙盒，但是苦于不会 Framework，而且学业繁忙，希望有大佬可以加入开发。