> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269895.htm)

> [原创] 批量检测 android app 的 so 中是否有 svc 调用

原理很检测对于 armv7 svc 0 对应的 00DF 二进制 调用号是在 r7 寄存器中  
armv8 svc 0 对应的 010000D4 二进制 调用号是在 x8 寄存器中，只要在 so 文件中的. text 段中找到相关的二进制字串就可以确定是否有 svc 调用，再向前看就可以看出 svc 的调用号是多少。不同人不同写法，会导致分析调用号多少不一样

 

![](https://bbs.pediy.com/upload/attach/202110/909269_KT7WRZS7AS446SA.png)  
上图这个 svc 调用就比较难分析出来  
而下图这个就比较简单。  
![](https://bbs.pediy.com/upload/attach/202110/909269_QJQC3T92DSKWQXU.png)

 

实现原理就是遍历文件夹下的每个 so 文件读取 elf 信息找出. text 代码范围，然后在这个里面找 010000D4 字串，然后向前去找 X8 的值 。  
svc 调用可以越过沙箱，libc 的 hook, 在一定程度上进行风控。

```
#只针对armv8 so进行扫描
# coding=utf-8
# coding=utf-8
import os
import re
 
import shutil
import binascii
import struct
import pathlib
from elftools.elf.elffile import  ELFFile
root_path =r'E:\svc_call_demo\app\build\outputs\apk\debug\app-debug\lib\arm64-v8a'
root_path =r'E:\svc_call_demo\app\build\outputs\apk\release\app-release\lib\arm64-v8a'
root_path =r'E:\weixin8015_32\lib\arm64-v8a\test'
#root_path =r'D:\working\android\crack_huifeng\src\main\lib\arm64-v8a'
sysCallTab = {}
sysCallTab[0] ="__NR_io_setup";
sysCallTab[1] ="__NR_io_destroy";
sysCallTab[2] ="__NR_io_submit";
sysCallTab[3] ="__NR_io_cancel";
sysCallTab[4] ="__NR_io_getevents";
sysCallTab[5] ="__NR_setxattr";
sysCallTab[6] ="__NR_lsetxattr";
sysCallTab[7] ="__NR_fsetxattr";
sysCallTab[8] ="__NR_getxattr";
sysCallTab[9] ="__NR_lgetxattr";
sysCallTab[10] ="__NR_fgetxattr";
sysCallTab[11] ="__NR_listxattr";
sysCallTab[12] ="__NR_llistxattr";
sysCallTab[13] ="__NR_flistxattr";
sysCallTab[14] ="__NR_removexattr";
sysCallTab[15] ="__NR_lremovexattr";
sysCallTab[16] ="__NR_fremovexattr";
sysCallTab[17] ="__NR_getcwd";
sysCallTab[18] ="__NR_lookup_dcookie";
sysCallTab[19] ="__NR_eventfd2";
sysCallTab[20] ="__NR_epoll_create1";
sysCallTab[21] ="__NR_epoll_ctl";
sysCallTab[22] ="__NR_epoll_pwait";
sysCallTab[23] ="__NR_dup";
sysCallTab[24] ="__NR_dup3";
sysCallTab[25] ="__NR3264_fcntl";
sysCallTab[26] ="__NR_inotify_init1";
sysCallTab[27] ="__NR_inotify_add_watch";
sysCallTab[28] ="__NR_inotify_rm_watch";
sysCallTab[29] ="__NR_ioctl";
sysCallTab[30] ="__NR_ioprio_set";
sysCallTab[31] ="__NR_ioprio_get";
sysCallTab[32] ="__NR_flock";
sysCallTab[33] ="__NR_mknodat";
sysCallTab[34] ="__NR_mkdirat";
sysCallTab[35] ="__NR_unlinkat";
sysCallTab[36] ="__NR_symlinkat";
sysCallTab[37] ="__NR_linkat";
sysCallTab[38] ="__NR_renameat";
sysCallTab[39] ="__NR_umount2";
sysCallTab[40] ="__NR_mount";
sysCallTab[41] ="__NR_pivot_root";
sysCallTab[42] ="__NR_nfsservctl";
sysCallTab[43] ="__NR3264_statfs";
sysCallTab[44] ="__NR3264_fstatfs";
sysCallTab[45] ="__NR3264_truncate";
sysCallTab[46] ="__NR3264_ftruncate";
sysCallTab[47] ="__NR_fallocate";
sysCallTab[48] ="__NR_faccessat";
sysCallTab[49] ="__NR_chdir";
sysCallTab[50] ="__NR_fchdir";
sysCallTab[51] ="__NR_chroot";
sysCallTab[52] ="__NR_fchmod";
sysCallTab[53] ="__NR_fchmodat";
sysCallTab[54] ="__NR_fchownat";
sysCallTab[55] ="__NR_fchown";
sysCallTab[56] ="__NR_openat";
sysCallTab[57] ="__NR_close";
sysCallTab[58] ="__NR_vhangup";
sysCallTab[59] ="__NR_pipe2";
sysCallTab[60] ="__NR_quotactl";
sysCallTab[61] ="__NR_getdents64";
sysCallTab[62] ="__NR3264_lseek";
sysCallTab[63] ="__NR_read";
sysCallTab[64] ="__NR_write";
sysCallTab[65] ="__NR_readv";
sysCallTab[66] ="__NR_writev";
sysCallTab[67] ="__NR_pread64";
sysCallTab[68] ="__NR_pwrite64";
sysCallTab[69] ="__NR_preadv";
sysCallTab[70] ="__NR_pwritev";
sysCallTab[71] ="__NR3264_sendfile";
sysCallTab[72] ="__NR_pselect6";
sysCallTab[73] ="__NR_ppoll";
sysCallTab[74] ="__NR_signalfd4";
sysCallTab[75] ="__NR_vmsplice";
sysCallTab[76] ="__NR_splice";
sysCallTab[77] ="__NR_tee";
sysCallTab[78] ="__NR_readlinkat";
sysCallTab[79] ="__NR3264_fstatat";
sysCallTab[80] ="__NR3264_fstat";
sysCallTab[81] ="__NR_sync";
sysCallTab[82] ="__NR_fsync";
sysCallTab[83] ="__NR_fdatasync";
sysCallTab[84] ="__NR_sync_file_range2";
sysCallTab[84] ="__NR_sync_file_range";
sysCallTab[85] ="__NR_timerfd_create";
sysCallTab[86] ="__NR_timerfd_settime";
sysCallTab[87] ="__NR_timerfd_gettime";
sysCallTab[88] ="__NR_utimensat";
sysCallTab[89] ="__NR_acct";
sysCallTab[90] ="__NR_capget";
sysCallTab[91] ="__NR_capset";
sysCallTab[92] ="__NR_personality";
sysCallTab[93] ="__NR_exit";
sysCallTab[94] ="__NR_exit_group";
sysCallTab[95] ="__NR_waitid";
sysCallTab[96] ="__NR_set_tid_address";
sysCallTab[97] ="__NR_unshare";
sysCallTab[98] ="__NR_futex";
sysCallTab[99] ="__NR_set_robust_list";
sysCallTab[100] ="__NR_get_robust_list";
sysCallTab[101] ="__NR_nanosleep";
sysCallTab[102] ="__NR_getitimer";
sysCallTab[103] ="__NR_setitimer";
sysCallTab[104] ="__NR_kexec_load";
sysCallTab[105] ="__NR_init_module";
sysCallTab[106] ="__NR_delete_module";
sysCallTab[107] ="__NR_timer_create";
sysCallTab[108] ="__NR_timer_gettime";
sysCallTab[109] ="__NR_timer_getoverrun";
sysCallTab[110] ="__NR_timer_settime";
sysCallTab[111] ="__NR_timer_delete";
sysCallTab[112] ="__NR_clock_settime";
sysCallTab[113] ="__NR_clock_gettime";
sysCallTab[114] ="__NR_clock_getres";
sysCallTab[115] ="__NR_clock_nanosleep";
sysCallTab[116] ="__NR_syslog";
sysCallTab[117] ="__NR_ptrace";
sysCallTab[118] ="__NR_sched_setparam";
sysCallTab[119] ="__NR_sched_setscheduler";
sysCallTab[120] ="__NR_sched_getscheduler";
sysCallTab[121] ="__NR_sched_getparam";
sysCallTab[122] ="__NR_sched_setaffinity";
sysCallTab[123] ="__NR_sched_getaffinity";
sysCallTab[124] ="__NR_sched_yield";
sysCallTab[125] ="__NR_sched_get_priority_max";
sysCallTab[126] ="__NR_sched_get_priority_min";
sysCallTab[127] ="__NR_sched_rr_get_interval";
sysCallTab[128] ="__NR_restart_syscall";
sysCallTab[129] ="__NR_kill";
sysCallTab[130] ="__NR_tkill";
sysCallTab[131] ="__NR_tgkill";
sysCallTab[132] ="__NR_sigaltstack";
sysCallTab[133] ="__NR_rt_sigsuspend";
sysCallTab[134] ="__NR_rt_sigaction";
sysCallTab[135] ="__NR_rt_sigprocmask";
sysCallTab[136] ="__NR_rt_sigpending";
sysCallTab[137] ="__NR_rt_sigtimedwait";
sysCallTab[138] ="__NR_rt_sigqueueinfo";
sysCallTab[139] ="__NR_rt_sigreturn";
sysCallTab[140] ="__NR_setpriority";
sysCallTab[141] ="__NR_getpriority";
sysCallTab[142] ="__NR_reboot";
sysCallTab[143] ="__NR_setregid";
sysCallTab[144] ="__NR_setgid";
sysCallTab[145] ="__NR_setreuid";
sysCallTab[146] ="__NR_setuid";
sysCallTab[147] ="__NR_setresuid";
sysCallTab[148] ="__NR_getresuid";
sysCallTab[149] ="__NR_setresgid";
sysCallTab[150] ="__NR_getresgid";
sysCallTab[151] ="__NR_setfsuid";
sysCallTab[152] ="__NR_setfsgid";
sysCallTab[153] ="__NR_times";
sysCallTab[154] ="__NR_setpgid";
sysCallTab[155] ="__NR_getpgid";
sysCallTab[156] ="__NR_getsid";
sysCallTab[157] ="__NR_setsid";
sysCallTab[158] ="__NR_getgroups";
sysCallTab[159] ="__NR_setgroups";
sysCallTab[160] ="__NR_uname";
sysCallTab[161] ="__NR_sethostname";
sysCallTab[162] ="__NR_setdomainname";
sysCallTab[163] ="__NR_getrlimit";
sysCallTab[164] ="__NR_setrlimit";
sysCallTab[165] ="__NR_getrusage";
sysCallTab[166] ="__NR_umask";
sysCallTab[167] ="__NR_prctl";
sysCallTab[168] ="__NR_getcpu";
sysCallTab[169] ="__NR_gettimeofday";
sysCallTab[170] ="__NR_settimeofday";
sysCallTab[171] ="__NR_adjtimex";
sysCallTab[172] ="__NR_getpid";
sysCallTab[173] ="__NR_getppid";
sysCallTab[174] ="__NR_getuid";
sysCallTab[175] ="__NR_geteuid";
sysCallTab[176] ="__NR_getgid";
sysCallTab[177] ="__NR_getegid";
sysCallTab[178] ="__NR_gettid";
sysCallTab[179] ="__NR_sysinfo";
sysCallTab[180] ="__NR_mq_open";
sysCallTab[181] ="__NR_mq_unlink";
sysCallTab[182] ="__NR_mq_timedsend";
sysCallTab[183] ="__NR_mq_timedreceive";
sysCallTab[184] ="__NR_mq_notify";
sysCallTab[185] ="__NR_mq_getsetattr";
sysCallTab[186] ="__NR_msgget";
sysCallTab[187] ="__NR_msgctl";
sysCallTab[188] ="__NR_msgrcv";
sysCallTab[189] ="__NR_msgsnd";
sysCallTab[190] ="__NR_semget";
sysCallTab[191] ="__NR_semctl";
sysCallTab[192] ="__NR_semtimedop";
sysCallTab[193] ="__NR_semop";
sysCallTab[194] ="__NR_shmget";
sysCallTab[195] ="__NR_shmctl";
sysCallTab[196] ="__NR_shmat";
sysCallTab[197] ="__NR_shmdt";
sysCallTab[198] ="__NR_socket";
sysCallTab[199] ="__NR_socketpair";
sysCallTab[200] ="__NR_bind";
sysCallTab[201] ="__NR_listen";
sysCallTab[202] ="__NR_accept";
sysCallTab[203] ="__NR_connect";
sysCallTab[204] ="__NR_getsockname";
sysCallTab[205] ="__NR_getpeername";
sysCallTab[206] ="__NR_sendto";
sysCallTab[207] ="__NR_recvfrom";
sysCallTab[208] ="__NR_setsockopt";
sysCallTab[209] ="__NR_getsockopt";
sysCallTab[210] ="__NR_shutdown";
sysCallTab[211] ="__NR_sendmsg";
sysCallTab[212] ="__NR_recvmsg";
sysCallTab[213] ="__NR_readahead";
sysCallTab[214] ="__NR_brk";
sysCallTab[215] ="__NR_munmap";
sysCallTab[216] ="__NR_mremap";
sysCallTab[217] ="__NR_add_key";
sysCallTab[218] ="__NR_request_key";
sysCallTab[219] ="__NR_keyctl";
sysCallTab[220] ="__NR_clone";
sysCallTab[221] ="__NR_execve";
sysCallTab[222] ="__NR3264_mmap";
sysCallTab[223] ="__NR3264_fadvise64";
sysCallTab[224] ="__NR_swapon";
sysCallTab[225] ="__NR_swapoff";
sysCallTab[226] ="__NR_mprotect";
sysCallTab[227] ="__NR_msync";
sysCallTab[228] ="__NR_mlock";
sysCallTab[229] ="__NR_munlock";
sysCallTab[230] ="__NR_mlockall";
sysCallTab[231] ="__NR_munlockall";
sysCallTab[232] ="__NR_mincore";
sysCallTab[233] ="__NR_madvise";
sysCallTab[234] ="__NR_remap_file_pages";
sysCallTab[235] ="__NR_mbind";
sysCallTab[236] ="__NR_get_mempolicy";
sysCallTab[237] ="__NR_set_mempolicy";
sysCallTab[238] ="__NR_migrate_pages";
sysCallTab[239] ="__NR_move_pages";
sysCallTab[240] ="__NR_rt_tgsigqueueinfo";
sysCallTab[241] ="__NR_perf_event_open";
sysCallTab[242] ="__NR_accept4";
sysCallTab[243] ="__NR_recvmmsg";
sysCallTab[244] ="__NR_arch_specific_syscall";
sysCallTab[260] ="__NR_wait4";
sysCallTab[261] ="__NR_prlimit64";
sysCallTab[262] ="__NR_fanotify_init";
sysCallTab[263] ="__NR_fanotify_mark";
sysCallTab[264] ="__NR_name_to_handle_at";
sysCallTab[265] ="__NR_open_by_handle_at";
sysCallTab[266] ="__NR_clock_adjtime";
sysCallTab[267] ="__NR_syncfs";
sysCallTab[268] ="__NR_setns";
sysCallTab[269] ="__NR_sendmmsg";
sysCallTab[270] ="__NR_process_vm_readv";
sysCallTab[271] ="__NR_process_vm_writev";
sysCallTab[272] ="__NR_kcmp";
sysCallTab[273] ="__NR_finit_module";
sysCallTab[274] ="__NR_sched_setattr";
sysCallTab[275] ="__NR_sched_getattr";
sysCallTab[276] ="__NR_renameat2";
sysCallTab[277] ="__NR_seccomp";
sysCallTab[278] ="__NR_getrandom";
sysCallTab[279] ="__NR_memfd_create";
sysCallTab[280] ="__NR_bpf";
sysCallTab[281] ="__NR_execveat";
sysCallTab[282] ="__NR_userfaultfd";
sysCallTab[283] ="__NR_membarrier";
sysCallTab[284] ="__NR_mlock2";
sysCallTab[285] ="__NR_copy_file_range";
sysCallTab[286] ="__NR_preadv2";
sysCallTab[287] ="__NR_pwritev2";
sysCallTab[288] ="__NR_pkey_mprotect";
sysCallTab[289] ="__NR_pkey_alloc";
sysCallTab[290] ="__NR_pkey_free";
sysCallTab[291] ="__NR_statx";
sysCallTab[292] ="__NR_io_pgetevents";
sysCallTab[293] ="__NR_rseq";
sysCallTab[294] ="__NR_kexec_file_load";
sysCallTab[403] ="__NR_clock_gettime64";
sysCallTab[404] ="__NR_clock_settime64";
sysCallTab[405] ="__NR_clock_adjtime64";
sysCallTab[406] ="__NR_clock_getres_time64";
sysCallTab[407] ="__NR_clock_nanosleep_time64";
sysCallTab[408] ="__NR_timer_gettime64";
sysCallTab[409] ="__NR_timer_settime64";
sysCallTab[410] ="__NR_timerfd_gettime64";
sysCallTab[411] ="__NR_timerfd_settime64";
sysCallTab[412] ="__NR_utimensat_time64";
sysCallTab[413] ="__NR_pselect6_time64";
sysCallTab[414] ="__NR_ppoll_time64";
sysCallTab[416] ="__NR_io_pgetevents_time64";
sysCallTab[417] ="__NR_recvmmsg_time64";
sysCallTab[418] ="__NR_mq_timedsend_time64";
sysCallTab[419] ="__NR_mq_timedreceive_time64";
sysCallTab[420] ="__NR_semtimedop_time64";
sysCallTab[421] ="__NR_rt_sigtimedwait_time64";
sysCallTab[422] ="__NR_futex_time64";
sysCallTab[423] ="__NR_sched_rr_get_interval_time64";
sysCallTab[424] ="__NR_pidfd_send_signal";
sysCallTab[425] ="__NR_io_uring_setup";
sysCallTab[426] ="__NR_io_uring_enter";
sysCallTab[427] ="__NR_io_uring_register";
sysCallTab[428] ="__NR_open_tree";
sysCallTab[429] ="__NR_move_mount";
sysCallTab[430] ="__NR_fsopen";
sysCallTab[431] ="__NR_fsconfig";
sysCallTab[432] ="__NR_fsmount";
sysCallTab[433] ="__NR_fspick";
sysCallTab[434] ="__NR_pidfd_open";
sysCallTab[435] ="__NR_clone3";
sysCallTab[436] ="__NR_syscalls";
 
 
def get_file_path(root_path,file_list,dir_list):
    #获取该目录下所有的文件名称和目录名称
    dir_or_files = os.listdir(root_path)
    for dir_file in dir_or_files:
        #获取目录或者文件的路径
        dir_file_path = os.path.join(root_path,dir_file)
        #判断该路径为文件还是路径
        if os.path.isdir(dir_file_path):
            dir_list.append(dir_file_path)
            #递归获取所有文件和目录的路径
            get_file_path(dir_file_path,file_list,dir_list)
        else:
            file_list.append(dir_file_path)
 
def alter(file, old_str, new_str):
    """
    替换文件中的字符串
    :param file:文件名
    :param old_str:就字符串
    :param new_str:新字符串
    :return:
    """
    try:
        file_data = ""
        f = open(file, 'rb').read()
        file_data=f.decode()
        find_info=False
        if old_str in file_data:
            file_data = file_data.replace(old_str, new_str)
            find_info=True
        if find_info:
            with open(file, "w",newline="") as f:
                f.write(file_data)
 
            print( file, "ok")
            #shutil.copy(file ,r"D:\pro\termux")
        else:
            #print(file, "error")
            pass
 
    except Exception as e:
        #print(file,"error")
        pass
 
 
extfun = lambda x: x
 
 
def read_file_hex(file_path):
    file_object = open(file_path, 'rb')
    file_object.seek(0, 0)
    hex_str = ''
    byte = file_object.read()
    if byte:
        for b in byte:
            b=b.to_bytes(length=1, byteorder='big', signed=False)
            hex_str += ('%02X' % extfun(ord(b)))
    file_object.close()
    return hex_str
 
 
def wirte_to_file(hex, file_path):
    fout = open(file_path, 'wb')
    fileLength=(int)(len(hex) / 2)
    for i in range( fileLength):
        x = int(hex[2 * i:2 * (i + 1)], 16)
        fout.write(struct.pack('B', extfun(x)))
    fout.close()
 
 
def hex_replace(hex, find_str, replace_str):
    return hex.replace(find_str, replace_str)
 
 
# 获取目录下的文件
def file_name(file_dir):
    for root, dirs, files in os.walk(file_dir):
        return (files)
 
# 获取目录下的目录
def file_dir(file_dir):
    for root, dirs, files in os.walk(file_dir):
        return (dirs)
 
# 获取后缀名
def file_extension(file):
    return os.path.splitext(file)[1]
 
# 获取后缀名
def str_to_hexStr(string):
    str_bin = string.encode('utf-8')
    return binascii.hexlify(str_bin).decode('utf-8').upper()
 
import time
 
start = time.time()
# hex_str=str_to_hexStr(r"com.termux/")
#root_path =  r'D:\working\AndroidLinux\ori_rom\extract\final'
#root_path = r'D:\working\android_linux\sermux\rom\from_rk3399\usr'
#root_path =r'D:\working\androidlinux\sermux\serumx-app\app\src\main\cpp\bootstrap-aarch64'
#            D:/working/AndroidLinux/sermux/rom/data/data/com.sermux/files/usr/etc/alternatives/vi|./bin/vi
#这个目录是从termux做好tar.gz文件解压缩后的romm目录
 
#root_path =r'E:\frida\AntiFrida-master\app\build\outputs\apk\debug\app-debug\lib\armeabi-v7a'
 
src_path = root_path
dst_path = root_path
 
#用来存放所有的文件路径
file_list = []
#用来存放所有的目录路径
dir_list = []
get_file_path(root_path,file_list,dir_list)
totalchangedfiles=0
for idx, file_path in enumerate(file_list):
    #alter(file_path, b'termux/', b'sermux/')
    if file_extension(file_path) == ".so":
        try:
            if("libdetection_based_tracker.so" in file_path):
                findfile=1
            #将文件转为十六进制字串，便于检索
            file_str = read_file_hex(file_path)
            # 获取文件大小
            filesize= os.path.getsize(file_path)
            # 获取文件行数，每32个字为一行，打印
            #linesize=int(filesize/32)-1
            # for i in range(0,linesize):
            #     text_list = re.findall(".{2}", file_str[i*32+0:i*32+32])
            #     new_text = " ".join(text_list)
            #     print(hex(i*16).replace("0x","")+"h",new_text)
            #wirte_to_file(file_str,file_path+".so")
 
 
            textStart = 0
            textEnd = 0
            FindtextEnd =False
            #读取elf文件信息，计算代码的地址与终止段 ,取section.name=='.text'后，马上取下一个就是textEnd
            #
            with open(file_path,'rb') as f:
                e=ELFFile(f)
                for section in e.iter_sections():
                    #打印Elf各段地址
                    #print(section['sh_addr'],hex(section['sh_addr']),section.name)
                    if FindtextEnd is True:
                        textEnd = section['sh_addr']
                        break
                    if(section.name=='.text'):
                        textStart=section['sh_addr']
                        FindtextEnd=True
 
 
            # hex_txt_file = file_path + ".txt"
            # with open(hex_txt_file, "w") as file:  # 只需要将之前的”w"改为“a"即可，代表追加内容
            #     file.write(file_str )
            #     print("generate",hex_txt_file)
 
 
            # sub = "00EF"
            # addr = [substr.start() for substr in re.finditer(sub, file_str)]
            # total_svc=0
            # for i in addr:
            #     if( i >= textStart and  i< textEnd):
            #         #之前有问题是因为地址没对齐，比如 E0 0E F1 转成字串时 E00EF1
 
            #         # 也包含了00EF,这就要用地址必须为偶数
            #         m=int(i/2)-6
            #         if(i % 8==0):
            #             total_svc = total_svc + 1
            #             print(total_svc,file_str[i-12:i+4],"  Func Name : needtocheck     addr : 0x%x" % (m ))
            # print(os.path.basename(file_path), "totoal find svc call ", total_svc)
 
 
            #sub="000000EF"
            sub = "010000D4"
            addr = [substr.start() for substr in re.finditer(sub, file_str)]
            total_svc = 0
 
            for i in addr:
                if( i >= textStart and  i< textEnd):
                    funid1 = int(int("0x"+file_str[i - 8:i - 6],16)/16)
                    funid2 = int("0x" + file_str[i - 6:i - 4], 16)
                    fun_id= int( (funid2*16+funid1 )/2)
                    m=int(i/2)-4
                    if (i % 8 == 0):
                        total_svc = total_svc + 1
                        if fun_id>0:
                            #还有一种异常 B8 FF 00 00   00 FF 02 1F  是两个命令的头尾
                            if(  fun_id  in sysCallTab):
                                print(file_str[i-8:i+8],hex(fun_id),"addr : 0x%.8x      Func Name : %s     " % (m,sysCallTab[fun_id] ))
                            else:
                                print(file_str[i - 8:i + 8], hex(fun_id),"addr : 0x%.8x       Func Name : need check again*********    " % (  m))
 
                        else:
                            print(file_str[i-8:i+8],hex(fun_id),"  Func Name : needtocheck     addr : 0x%x" % (m ))
 
            if( total_svc >0):
                print(os.path.basename(file_path), "elf infor ")
                print(os.path.basename(file_path), ".text start ", hex(textStart), ".text end ", hex(textEnd))
                print(os.path.basename(file_path), "find svc call ", total_svc)
                print("——————————————————————————————\n")
 
 
        except Exception as e:
            print(file_path, "--------------------------->error")
            pass
 
 
        totalchangedfiles = totalchangedfiles + 1
 
        finalpath = dst_path + file_path.replace(root_path, "")
        finalDir = os.path.dirname(finalpath)
        #finalDir = pathlib.Path(finalpath)
        if  os.path.exists(finalDir):
            pass
        else:
            os.makedirs(finalDir)
            pass
        #shutil.copy(file_path, finalpath)
            #print(file_path, "--------------------------->", finalpath)
            #   pass
 
 
print( "\n--------------------------->finished<--------------------------")
end = time.time()
running_time = end-start

```

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 2 小时前 被 failure114 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#源码分析](forum-161-1-127.htm)