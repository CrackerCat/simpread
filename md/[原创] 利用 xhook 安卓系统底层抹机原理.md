> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265351.htm)

先上 github 地址 [https://github.com/iqiyi/xhook](https://github.com/iqiyi/xhook)

大概原理是：

*   先读取 / proc/self/maps 文件内容
    
*   正则匹配找到 so 文件路径和加载基址，
    
*   解析 elf 格式找到要 hook 的函数的地址替换成自己指定的函数地址
    

```
//xh_core.c
static void xh_core_refresh_impl()    
{    
char                     line[512];    
FILE                    *fp;    
uintptr_t                base_addr;    
char                     perm[5];    
unsigned long            offset;    
int                      pathname_pos;    
char                    *pathname;    
size_t                   pathname_len;    
xh_core_map_info_t      *mi, *mi_tmp;    
xh_core_map_info_t       mi_key;    
xh_core_hook_info_t     *hi;    
xh_core_ignore_info_t   *ii;    
int                      match;    
xh_core_map_info_tree_t  map_info_refreshed = RB_INITIALIZER(&map_info_refreshed);    
if(NULL == (fp = fopen("/proc/self/maps", "r")))    
{    
XH_LOG_ERROR("fopen /proc/self/maps failed");    
return;    
}    
while(fgets(line, sizeof(line), fp))    
{    
if(sscanf(line, "%"PRIxPTR"-%*lx %4s %lx %*x:%*x %*d%n", &base_addr, perm, &offset, &pathname_pos) != 3) continue;    
//check permission    
if(perm[0] != 'r') continue;    
if(perm[3] != 'p') continue; //do not touch the shared memory    
//check offset    
//    
//We are trying to find ELF header in memory.    
//It can only be found at the beginning of a mapped memory regions    
//whose offset is 0.    
if(0 != offset) continue;    
//get pathname    
while(isspace(line[pathname_pos]) && pathname_pos < (int)(sizeof(line) - 1))    
pathname_pos += 1;    
if(pathname_pos >= (int)(sizeof(line) - 1)) continue;    
pathname = line + pathname_pos;    
pathname_len = strlen(pathname);    
if(0 == pathname_len) continue;    
if(pathname[pathname_len - 1] == '\n')    
{    
pathname[pathname_len - 1] = '\0';    
pathname_len -= 1;    
}    
if(0 == pathname_len) continue;    
if('[' == pathname[0]) continue;    
//check pathname    
//if we need to hook this elf?    
match = 0;    
TAILQ_FOREACH(hi, &xh_core_hook_info, link) //find hook info    
{    
if(0 == regexec(&(hi->pathname_regex), pathname, 0, NULL, 0))    
{    
TAILQ_FOREACH(ii, &xh_core_ignore_info, link) //find ignore info    
{    
if(0 == regexec(&(ii->pathname_regex), pathname, 0, NULL, 0))    
{    
if(NULL == ii->symbol)    
goto check_finished;    
if(0 == strcmp(ii->symbol, hi->symbol))    
goto check_continue;    
}    
}    
match = 1;    
check_continue:    
break;    
}    
}    
check_finished:    
if(0 == match) continue;    
//check elf header format    
//We are trying to do ELF header checking as late as possible.    
if(0 != xh_core_check_elf_header(base_addr, pathname)) continue;    
//check existed map item    
mi_key.pathname = pathname;    
if(NULL != (mi = RB_FIND(xh_core_map_info_tree, &xh_core_map_info, &mi_key)))    
{    
//exist    
RB_REMOVE(xh_core_map_info_tree, &xh_core_map_info, mi);    
//repeated?    
//We only keep the first one, that is the real base address    
if(NULL != RB_INSERT(xh_core_map_info_tree, &map_info_refreshed, mi))    
{    
#if XH_CORE_DEBUG    
XH_LOG_DEBUG("repeated map info when update: %s", line);    
#endif    
free(mi->pathname);    
free(mi);    
continue;    
}    
//re-hook if base_addr changed    
if(mi->base_addr != base_addr)    
{    
mi->base_addr = base_addr;    
xh_core_hook(mi);    
}    
}    
else    
{    
//not exist, create a new map info    
if(NULL == (mi = (xh_core_map_info_t *)malloc(sizeof(xh_core_map_info_t)))) continue;    
if(NULL == (mi->pathname = strdup(pathname)))    
{    
free(mi);    
continue;    
}    
mi->base_addr = base_addr;    
//repeated?    
//We only keep the first one, that is the real base address    
if(NULL != RB_INSERT(xh_core_map_info_tree, &map_info_refreshed, mi))    
{    
#if XH_CORE_DEBUG    
XH_LOG_DEBUG("repeated map info when create: %s", line);    
#endif    
free(mi->pathname);    
free(mi);    
continue;    
}    
//hook    
xh_core_hook(mi); //hook    
}    
}    
fclose(fp);    
//free all missing map item, maybe dlclosed?    
RB_FOREACH_SAFE(mi, xh_core_map_info_tree, &xh_core_map_info, mi_tmp)    
{    
#if XH_CORE_DEBUG    
XH_LOG_DEBUG("remove missing map info: %s", mi->pathname);    
#endif    
RB_REMOVE(xh_core_map_info_tree, &xh_core_map_info, mi);    
if(mi->pathname) free(mi->pathname);    
free(mi);    
}    
//save the new refreshed map info tree    
xh_core_map_info = map_info_refreshed;    
XH_LOG_INFO("map refreshed");    
#if XH_CORE_DEBUG    
RB_FOREACH(mi, xh_core_map_info_tree, &xh_core_map_info)    
XH_LOG_DEBUG("  %"PRIxPTR" %s\n", mi->base_addr, mi->pathname);    
#endif    
}

```

地址替换通过 PLT 表, 详细原理 [https://zhuanlan.zhihu.com/p/36426206](https://zhuanlan.zhihu.com/p/36426206)。

```
//xh_elf.c
int xh_elf_hook(xh_elf_t *self, const char *symbol, void *new_func, void **old_func)
{
    uint32_t                        symidx;
    void                           *rel_common;
    xh_elf_plain_reloc_iterator_t   plain_iter;
    xh_elf_packed_reloc_iterator_t  packed_iter;
    int                             found;
    int                             r;
 
    if(NULL == self->pathname)
    {
        XH_LOG_ERROR("not inited\n");
        return XH_ERRNO_ELFINIT; //not inited?
    }
 
    if(NULL == symbol || NULL == new_func) return XH_ERRNO_INVAL;
 
    XH_LOG_INFO("hooking %s in %s\n", symbol, self->pathname);
     
    //find symbol index by symbol name
    if(0 != (r = xh_elf_find_symidx_by_name(self, symbol, &symidx))) return 0;
     
    //replace for .rel(a).plt
    if(0 != self->relplt)
    {
        xh_elf_plain_reloc_iterator_init(&plain_iter, self->relplt, self->relplt_sz, self->is_use_rela);
        while(NULL != (rel_common = xh_elf_plain_reloc_iterator_next(&plain_iter)))
        {
            if(0 != (r = xh_elf_find_and_replace_func(self,
                                                      (self->is_use_rela ? ".rela.plt" : ".rel.plt"), 1,
                                                      symbol, new_func, old_func,
                                                      symidx, rel_common, &found))) return r;
            if(found) break;
        }
    }
 
    //replace for .rel(a).dyn
    if(0 != self->reldyn)
    {
        xh_elf_plain_reloc_iterator_init(&plain_iter, self->reldyn, self->reldyn_sz, self->is_use_rela);
        while(NULL != (rel_common = xh_elf_plain_reloc_iterator_next(&plain_iter)))
        {
            if(0 != (r = xh_elf_find_and_replace_func(self,
                                                      (self->is_use_rela ? ".rela.dyn" : ".rel.dyn"), 0,
                                                      symbol, new_func, old_func,
                                                      symidx, rel_common, NULL))) return r;
        }
    }
 
    //replace for .rel(a).android
    if(0 != self->relandroid)
    {
        xh_elf_packed_reloc_iterator_init(&packed_iter, self->relandroid, self->relandroid_sz, self->is_use_rela);
        while(NULL != (rel_common = xh_elf_packed_reloc_iterator_next(&packed_iter)))
        {
            if(0 != (r = xh_elf_find_and_replace_func(self,
                                                      (self->is_use_rela ? ".rela.android" : ".rel.android"), 0,
                                                      symbol, new_func, old_func,
                                                      symidx, rel_common, NULL))) return r;
        }
    }
     
    return 0;
}

```

抹机软件通过 hook 一些底层函数达到抹机的目的，其中经常用到的函数如下：

```
int access(const char *pathname, int mode);//判断文件能否访问，一般判断是否存在root文件
int open(const char *pathname, int flags);//打开一些设备配置文件
FILE* fopen(const char *filename, const char *mode);//打开一些设备配置文件
prop_info* __system_property_find(const char* name);//判断build.prop等prop文件的key值存在否
int __system_property_read(const prop_info *pi, char *name, char *value);//同等android.os.SystemProperties.get("xxxx")
int __system_property_get(const char *name, char *value);//同等android.os.SystemProperties.get("xxxx")
FILE* popen( const char *command , const char *type );//执行shell命令并读取结果返回一个文件指针

```

实例代码如下：

```
#include  #include  #include  #include  #include  #include  #include  #include "xhook/src/xhook.h"
 
int (*oldaccess)(const char *pathname, int mode);
int (*oldopen)(const char *pathname, int flags);
FILE* (*oldfopen)(const char *filename, const char *mode);
FILE* (*oldpopen)( const char *command , const char *type );
prop_info* (*old_system_property_find)(const char* name);
int (*old_system_property_get)(const char *name, char *value);
int (*old_system_property_read)(const prop_info *pi, char *name, char *value);
 
 
static int new_access(const char * pathname, int mode){
    __android_log_print(3, "Nativehook", "Nativehook_access unhook %s %d", pathname, mode);
    return access(pathname,mode);
}
 
 
static int new_open(const char * pathname, int flags,...){
    __android_log_print(3, "Nativehook", "Nativehook_open unhook %s %d", pathname, flags);
    return oldopen(pathname,flags);
}
 
static FILE *new_fopen(const char *filename, const char *mode){
    __android_log_print(3, "Nativehook", "Nativehook_fopen unhook %s %s", filename, mode);
    return oldfopen(filename,mode);
}
 
static int new_system_property_get(const char *name, char *value){
    __android_log_print(3, "Nativehook", "Nativehook_system_property_get unhook %s %s",name,value);
    return old_system_property_get(name,value);
}
 
static int new_system_property_read(const prop_info *pi, char *name, char *value) {
    __android_log_print(3, "Nativehook", "Nativehook_system_property_read unhook %s %s",name,value);
    return old_system_property_read(pi,name,value);
}
 
const prop_info* new_system_property_find(const char* name){
    __android_log_print(3, "Nativehook", "Nativehook_system_property_find unhook %s",name);
    return old_system_property_find(name);
}
 
static FILE * newpopen(const char *command ,const char *type ){
    __android_log_print(3, "Nativehook", "Nativehook_popen unhook %s %s",command,type);
    return oldpopen(command,type);
}
 
void Java_com_ayona333_fix_party_NativeHook_start(JNIEnv* env, jobject obj)
{
    (void)env;
    (void)obj;
 
    xhook_ignore(".*/libnativehook.so$", NULL);//屏蔽hook自己
 
    xhook_register("/data/.*\\.so$", "access",new_access ,(void**)(&oldaccess));
 
    xhook_register("/data/.*\\.so$", "open", new_open,(void**)(&oldopen));
 
    xhook_register("/data/data.*\\.so$", "fopen",new_fopen,(void**)(&oldfopen));
 
    xhook_register("/data/.*\\.so$", "__system_property_find",new_system_property_find,(void**)(&old_system_property_find));
 
    xhook_register("/data/.*\\.so$", "__system_property_get",new_system_property_get,(void**)(&old_system_property_get));
 
    xhook_register("/data/.*\\.so$", "__system_property_read",new_system_property_read,(void**)(&old_system_property_read));
 
    xhook_register("/data/.*\\.so", "popen",newpopen,(void**)(&oldpopen));
 
    xhook_refresh(1);
}
 
jint Java_com_ayona333_fix_party_NativeHook_refresh(JNIEnv* env, jobject obj,jboolean flag)
{
    return xhook_refresh(flag);
}
 
void Java_com_ayona333_fix_party_NativeHook_clear(JNIEnv* env, jobject obj)
{
    xhook_clear();
}
 
void Java_com_ayona333_fix_party_NativeHook_enableDebug(JNIEnv* env, jobject obj,jboolean flag)
{
    xhook_enable_debug(flag);
}
 
void Java_com_ayona333_fix_party_NativeHook_enableSigSegvProtection(JNIEnv* env, jobject obj,jboolean flag)
{
    xhook_enable_sigsegv_protection(flag);
} 
```

大概原理明白了以后运用起来比较简单，实际场景一般跟 xposed 等 hook 框架配合，拦截需要抹机的 APP 进程后读取该 APP 的 / proc/self/maps 文件、然后进行 native 层的 hook。

或抽出 so 文件，然后嵌入到自己写 APP 里，配合 xhook 进行要抹机的 so 文件，绕过 native 层的设备指纹检测或者其他操作。

如果配合其他 inlineHook 框架的话能达到全 hook 的目的。

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)

最后于 1 天前 被 AyonA333 编辑 ，原因：