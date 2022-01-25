> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [codeshare.frida.re](https://codeshare.frida.re/@FrenchYeti/android-arm64-strace/)

> (function(){function r(e,n,t){function o(i,f){if(!n[i]){if(!e[i]){var c="function"==typeof require&&r......

(function(){function r(e,n,t){function o(i,f){if(!n[i]){if(!e[i]){var c="function"==typeof require&&require;if(!f&&c)return c(i,!0);if(u)return u(i,!0);var a=new Error("Cannot find module'"+i+"'");throw a.code="MODULE_NOT_FOUND",a}var p=n[i]={exports:{}};e[i][0].call(p.exports,function(r){var n=e[i][1][r];return o(n||r)},p,p.exports,r,e,n,t)}return n[i].exports}for(var u="function"==typeof require&&require,i=0;i<t.length;i++)o(t[i]);return o}return r})()({1:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

const LinuxArm64InterruptorFactory_1 = require("./src/arch/LinuxArm64InterruptorFactory");

const Interruptors = {

LinuxArm64: function (pOptions) {

return new LinuxArm64InterruptorFactory_1.LinuxArm64InterruptorFactory(pOptions);

}

};

exports.default = Interruptors;

},{"./src/arch/LinuxArm64InterruptorFactory":4}],2:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.X = exports.E = exports.S = exports.I = exports.AT_ = exports.MFD = exports.PROT_ = exports.MNT_ = exports.MAP_ = exports.MCL_ = exports.MS_ = exports.PR_ = exports.MADV_ = exports.PTRACE_ = exports.O_ = exports.K = void 0;

exports.K = {

P_ALL: [0],

P_PID: [1],

P_PGID: [2],

P_PIDFD: [3]

};

exports.O_ = {

O_ACCMODE: 0o00000003,

O_RDONLY: 0o0000000,

O_WRONLY: 0o0000001,

O_RDWR: 0o0000002,

O_CREAT: 0o0000100,

O_EXCL: 0o0000200,

O_NOCTTY: 0o0000400,

O_TRUNC: 0o0001000,

O_APPEND: 0o0002000,

O_NONBLOCK: 0o0004000,

O_DSYNC: 0o0010000,

O_ASYNC: 0o0020000,

O_DIRECT: 0o0040000,

O_LARGEFILE: 0o0100000,

O_DIRECTORY: 0o0200000,

O_NOFOLLOW: 0o0400000,

O_NOATIME: 0o1000000,

O_CLOEXEC: 0o2000000,

O_PATH: 0o10000000,

O_TMPFILE: 0o20040000

};

exports.PTRACE_ = {

PTRACE_TRACEME: [0],

PTRACE_PEEKTEXT: [1],

PTRACE_PEEKDATA: [2],

PTRACE_PEEKUSR: [3],

PTRACE_POKETEXT: [4],

PTRACE_POKEDATA: [5],

PTRACE_POKEUSR: [6],

PTRACE_CONT: [7],

PTRACE_KILL: [8],

PTRACE_SINGLESTEP: [9],

PTRACE_ATTACH: [16],

PTRACE_DETACH: [17],

PTRACE_SYSCALL: [24],

PTRACE_SETOPTIONS: [0x4200],

PTRACE_GETEVENTMSG: [0x4201],

PTRACE_GETSIGINFO: [0x4202],

PTRACE_SETSIGINFO: [0x4203],

PTRACE_GETREGSET: [0x4204],

PTRACE_SETREGSET: [0x4205],

PTRACE_SEIZE: [0x4206],

PTRACE_INTERRUPT: [0x4207],

PTRACE_LISTEN: [0x4208],

PTRACE_PEEKSIGINFO: [0x4209]

};

exports.MADV_ = {

MADV_NORMAL: [0],

MADV_RANDOM: [1],

MADV_SEQUENTIAL: [2],

MADV_WILLNEED: [3],

MADV_DONTNEED: [4],

MADV_FREE: [8],

MADV_REMOVE: [9],

MADV_DONTFORK: [10],

MADV_DOFORK: [11],

MADV_HWPOISON: [100],

MADV_SOFT_OFFLINE: [101],

MADV_MERGEABLE: [12],

MADV_UNMERGEABLE: [13],

MADV_HUGEPAGE: [14],

MADV_NOHUGEPAGE: [15],

MADV_DONTDUMP: [16],

MADV_DODUMP: [17],

MADV_WIPEONFORK: [18],

MADV_KEEPONFORK: [19],

MADV_COLD: [20],

MADV_PAGEOUT: [21]

};

exports.PR_ = {

OPT: {

PR_CAP_AMBIENT: [47],

PR_CAPBSET_READ: [23],

PR_CAPBSET_DROP: [24],

PR_SET_CHILD_SUBREAPER: [36],

PR_GET_CHILD_SUBREAPER: [37],

PR_SET_PDEATHSIG: [1],

PR_GET_PDEATHSIG: [2],

PR_GET_DUMPABLE: [3],

PR_SET_DUMPABLE: [4],

PR_GET_UNALIGN: [5],

PR_SET_UNALIGN: [6],

PR_GET_KEEPCAPS: [7],

PR_SET_KEEPCAPS: [8],

PR_GET_FPEMU: [9],

PR_SET_FPEMU: [10],

PR_GET_FPEXC: [11],

PR_SET_FPEXC: [12],

PR_GET_TIMING: [13],

PR_SET_TIMING: [14],

PR_SET_NAME: [15],

PR_GET_NAME: [16],

PR_GET_ENDIAN: [19],

PR_SET_ENDIAN: [20],

PR_GET_SECCOMP: [21],

PR_SET_SECCOMP: [22],

PR_GET_TSC: [25],

PR_SET_TSC: [26],

PR_GET_SECUREBITS: [27],

PR_SET_SECUREBITS: [28],

PR_SET_TIMERSLACK: [29],

PR_GET_TIMERSLACK: [30],

PR_SET_PTRACER: [0x59616d61],

PR_SET_PTRACER_ANY: [(0xffffffffffffffff - 1)],

PR_SET_NO_NEW_PRIVS: [38],

PR_GET_NO_NEW_PRIVS: [39],

PR_GET_TID_ADDRESS: [40],

PR_SET_THP_DISABLE: [41],

PR_GET_THP_DISABLE: [42],

PR_SET_IO_FLUSHER: [57],

PR_GET_IO_FLUSHER: [58],

PR_SET_SYSCALL_USER_DISPATCH: [59],

PR_SET_VMA: [0x53564d41],

PR_SET_VMA_ANON_NAME: [0],

PR_SET_TAGGED_ADDR_CTRL: [55],

PR_GET_TAGGED_ADDR_CTRL: [56],

PR_SET_MM: [35],

PR_SET_FP_MODE: [45],

PR_GET_FP_MODE: [46],

PR_GET_SPECULATION_CTRL: [52],

PR_SET_SPECULATION_CTRL: [53],

},

CAP: {

PR_CAP_AMBIENT_IS_SET: [1],

PR_CAP_AMBIENT_RAISE: [2],

PR_CAP_AMBIENT_LOWER: [3],

PR_CAP_AMBIENT_CLEAR_ALL: [4],

},

UNALIGN: {

PR_UNALIGN_NOPRINT: [1],

PR_UNALIGN_SIGBUS: [2],

},

FPEMU: {

PR_FPEMU_NOPRINT: [1],

PR_FPEMU_SIGFPE: [2],

},

FP: {

PR_FP_EXC_SW_ENABLE: [0x80],

PR_FP_EXC_DIV: [0x010000],

PR_FP_EXC_OVF: [0x020000],

PR_FP_EXC_UND: [0x040000],

PR_FP_EXC_RES: [0x080000],

PR_FP_EXC_INV: [0x100000],

PR_FP_EXC_DISABLED: [0],

PR_FP_EXC_NONRECOV: [1],

PR_FP_EXC_ASYNC: [2],

PR_FP_EXC_PRECISE: [3],

PR_FP_MODE_FR: [(1 << 0)],

PR_FP_MODE_FRE: [(1 << 1)],

},

TIMING: {

PR_TIMING_STATISTICAL: [0],

PR_TIMING_TIMESTAMP: [1],

},

ENDIAN: {

PR_ENDIAN_BIG: [0],

PR_ENDIAN_LITTLE: [1],

PR_ENDIAN_PPC_LITTLE: [2],

},

TSC: {

PR_TSC_ENABLE: [1],

PR_TSC_SIGSEGV: [2],

},

TASK: {

PR_TASK_PERF_EVENTS_DISABLE: [31],

PR_TASK_PERF_EVENTS_ENABLE: [32],

},

MCE: {

PR_MCE_KILL: [33],

PR_MCE_KILL_CLEAR: [0],

PR_MCE_KILL_SET: [1],

PR_MCE_KILL_LATE: [0],

PR_MCE_KILL_EARLY: [1],

PR_MCE_KILL_DEFAULT: [2],

PR_MCE_KILL_GET: [34],

},

MM: {

PR_SET_MM_START_CODE: [1],

PR_SET_MM_END_CODE: [2],

PR_SET_MM_START_DATA: [3],

PR_SET_MM_END_DATA: [4],

PR_SET_MM_START_STACK: [5],

PR_SET_MM_START_BRK: [6],

PR_SET_MM_BRK: [7],

PR_SET_MM_ARG_START: [8],

PR_SET_MM_ARG_END: [9],

PR_SET_MM_ENV_START: [10],

PR_SET_MM_ENV_END: [11],

PR_SET_MM_AUXV: [12],

PR_SET_MM_EXE_FILE: [13],

PR_SET_MM_MAP: [14],

PR_SET_MM_MAP_SIZE: [15]

},

MPX: {

PR_MPX_ENABLE_MANAGEMENT: [43],

PR_MPX_DISABLE_MANAGEMENT: [44],

},

SVE: {

PR_SVE_SET_VL: [50],

PR_SVE_SET_VL_ONEXEC: [(1 << 18)],

PR_SVE_GET_VL: [51],

PR_SVE_VL_LEN_MASK: [0xffff],

PR_SVE_VL_INHERIT: [(1 << 17)],

},

SPEC: {

PR_SPEC_STORE_BYPASS: [0],

PR_SPEC_INDIRECT_BRANCH: [1],

PR_SPEC_NOT_AFFECTED: [0],

PR_SPEC_PRCTL: [(1 << 0)],

PR_SPEC_ENABLE: [(1 << 1)],

PR_SPEC_DISABLE: [(1 << 2)],

PR_SPEC_FORCE_DISABLE: [(1 << 3)],

PR_SPEC_DISABLE_NOEXEC: [(1 << 4)],

},

PAC: {

PR_PAC_RESET_KEYS: [54],

PR_PAC_APIAKEY: [(1 << 0)],

PR_PAC_APIBKEY: [(1 << 1)],

PR_PAC_APDAKEY: [(1 << 2)],

PR_PAC_APDBKEY: [(1 << 3)],

PR_PAC_APGAKEY: [(1 << 4)],

},

TAGGED: {

PR_TAGGED_ADDR_ENABLE: [(1 << 0)],

},

MTE: {

PR_MTE_TCF_SHIFT: [1],

PR_MTE_TAG_SHIFT: [3],

PR_MTE_TCF_NONE: [(0 << 1)],

PR_MTE_TCF_SYNC: [(1 << 1)],

PR_MTE_TCF_ASYNC: [(2 << 1)],

PR_MTE_TCF_MASK: [(3 << 1)],

PR_MTE_TAG_MASK: [(0xffff << 3)],

},

SYS: {

PR_SYS_DISPATCH_OFF: [0],

PR_SYS_DISPATCH_ON: [1],

},

SYSCALL: {

SYSCALL_DISPATCH_FILTER_ALLOW: [0],

SYSCALL_DISPATCH_FILTER_BLOCK: [1]

}

};

exports.MS_ = {

MS_ASYNC: 1,

MS_INVALIDATE: 2,

MS_SYNC: 4

};

exports.MCL_ = {

MCL_CURRENT: 1,

MCL_FUTURE: 2,

MCL_ONFAULT: 4

};

exports.MAP_ = {

MAP_SHARED: [0x01],

MAP_PRIVATE: [0x02],

MAP_SHARED_VALIDATE: [0x03],

MAP_FIXED: [0x10],

MAP_ANONYMOUS: [0x20],

MAP_GROWSDOWN: [0x0100],

MAP_DENYWRITE: [0x0800],

MAP_EXECUTABLE: [0x1000],

MAP_LOCKED: [0x2000],

MAP_NORESERVE: [0x4000],

};

exports.MNT_ = {

MNT_FORCE: 1,

MNT_DETACH: 2,

MNT_EXPIRE: 4,

UMOUNT_NOFOLLOW: 8,

};

const PROT_NONE = 0;

exports.PROT_ = {

PROT_READ: 1,

PROT_WRITE: 2,

PROT_EXEC: 4,

PROT_SEM: 8,

PROT_GROWSDOWN: 0x01000000,

PROT_GROWSUP: 0x02000000

};

exports.MFD = {

MFD_CLOEXEC: 1,

MFD_ALLOW_SEALING: 2,

MFD_HUGETLB: 4,

};

exports.AT_ = {

AT_FDCWD: -100,

AT_SYMLINK_NOFOLLOW: [0x100],

AT_EACCESS: [0x200],

AT_REMOVEDIR: [0x200],

AT_SYMLINK_FOLLOW: [0x400],

AT_NO_AUTOMOUNT: [0x800],

AT_EMPTY_PATH: [0x1000],

AT_STATX_SYNC_TYPE: [0x6000],

AT_STATX_SYNC_AS_STAT: [0x0000],

AT_STATX_FORCE_SYNC: [0x2000],

AT_STATX_DONT_SYNC: [0x4000],

AT_RECURSIVE: [0x8000],

};

const F_ = {

F_DUPFD: [0],

F_GETFD: [1],

F_SETFD: [2],

F_GETFL: [3],

F_SETFL: [4],

F_SETOWN: [8],

F_GETOWN: [9],

F_SETSIG: [10],

F_GETSIG: [11],

F_GETLK: [12],

F_SETLK: [13],

F_SETLKW: [14],

F_SETOWN_EX: [15],

F_GETOWN_EX: [16],

F_GETOWNER_UIDS: [17]

};

function stringifyBitmap(val, flags) {

let s = "";

for (const f in flags) {

if ((val & flags[f]) == flags[f])

s += (s.length > 0 ? "|" : "") + f;

}

return s;

}

exports.I = {

KILL_FROM: function (ctx) {

const f = ctx.x0.toInt32();

if (f > 0) {

return f + "(target process)";

}

else if (f < 0) {

return f + "(all authorized processes)";

}

else {

return f + "(all processes from process group of calling process)";

}

}

};

exports.S = {

SIGHUP: [1],

SIGINT: [2],

SIGQUIT: [3],

SIGILL: [4],

SIGTRAP: [5],

SIGABRT: [6],

SIGIOT: [6],

SIGBUS: [7],

SIGFPE: [8],

SIGKILL: [9],

SIGUSR1: [10],

SIGSEGV: [11],

SIGUSR2: [12],

SIGPIPE: [13],

SIGALRM: [14],

SIGTERM: [15],

SIGSTKFLT: [16],

SIGCHLD: [17],

SIGCONT: [18],

SIGSTOP: [19],

SIGTSTP: [20],

SIGTTIN: [21],

SIGTTOU: [22],

SIGURG: [23],

SIGXCPU: [24],

SIGXFSZ: [25],

SIGVTALRM: [26],

SIGPROF: [27],

SIGWINCH: [28],

SIGIO: [29],

SIGPWR: [30],

SIGSYS: [31],

SIGUNUSED: [31],

SIGRTMIN: [32],

SIGRTMAX: [64],

MINSIGSTKSZ: [2048],

SIGSTKSZ: [8192],

};

exports.E = {

EPERM: [1, "Not super-user"],

ENOENT: [2, "No such file or directory"],

ESRCH: [3, "No such process"],

EINTR: [4, "Interrupted system call"],

EIO: [5, "I/O error"],

ENXIO: [6, "No such device or address"],

E2BIG: [7, "Arg list too long"],

ENOEXEC: [8, "Exec format error"],

EBADF: [9, "Bad file number"],

ECHILD: [10, "No children"],

EAGAIN: [11, "No more processes"],

ENOMEM: [12, "Not enough core"],

EACCES: [13, "Permission denied"],

EFAULT: [14, "Bad address"],

ENOTBLK: [15, "Block device required"],

EBUSY: [16, "Mount device busy"],

EEXIST: [17, "File exists"],

EXDEV: [18, "Cross-device link"],

ENODEV: [19, "No such device"],

ENOTDIR: [20, "Not a directory"],

EISDIR: [21, "Is a directory"],

EINVAL: [22, "Invalid argument"],

ENFILE: [23, "Too many open files in system"],

EMFILE: [24, "Too many open files"],

ENOTTY: [25, "Not a typewriter"],

ETXTBSY: [26, "Text file busy"],

EFBIG: [27, "File too large"],

ENOSPC: [28, "No space left on device"],

ESPIPE: [29, "Illegal seek"],

EROFS: [30, "Read only file system"],

EMLINK: [31, "Too many links"],

EPIPE: [32, "Broken pipe"],

EDOM: [33, "Math arg out of domain of func"],

ERANGE: [34, "Math result not representable"],

ENOMSG: [35, "No message of desired type"],

EIDRM: [36, "Identifier removed"],

ECHRNG: [37, "Channel number out of range"],

EL2NSYNC: [38, "Level 2 not synchronized"],

EL3HLT: [39, "Level 3 halted"],

EL3RST: [40, "Level 3 reset"],

ELNRNG: [41, "Link number out of range"],

EUNATCH: [42, "Protocol driver not attached"],

ENOCSI: [43, "No CSI structure available"],

EL2HLT: [44, "Level 2 halted"],

EDEADLK: [45, "Deadlock condition"],

ENOLCK: [46, "No record locks available"],

EBADE: [50, "Invalid exchange"],

EBADR: [51, "Invalid request descriptor"],

EXFULL: [52, "Exchange full"],

ENOANO: [53, "No anode"],

EBADRQC: [54, "Invalid request code"],

EBADSLT: [55, "Invalid slot"],

EDEADLOCK: [56, "File locking deadlock error"],

EBFONT: [57, "Bad font file fmt"],

ENOSTR: [60, "Device not a stream"],

ENODATA: [61, "No data (for no delay io)"],

ETIME: [62, "Timer expired"],

ENOSR: [63, "Out of streams resources"],

ENONET: [64, "Machine is not on the network"],

ENOPKG: [65, "Package not installed"],

EREMOTE: [66, "The object is remote"],

ENOLINK: [67, "The link has been severed"],

EADV: [68, "Advertise error"],

ESRMNT: [69, "Srmount error"],

ECOMM: [70, "Communication error on send"],

EPROTO: [71, "Protocol error"],

EMULTIHOP: [74, "Multihop attempted"],

ELBIN: [75, "Inode is remote (not really error)"],

EDOTDOT: [76, "Cross mount point (not really error)"],

EBADMSG: [77, "Trying to read unreadable message"],

EFTYPE: [79, "Inappropriate file type or format"],

ENOTUNIQ: [80, "Given log. name not unique"],

EBADFD: [81, "f.d. invalid for this operation"],

EREMCHG: [82, "Remote address changed"],

ELIBACC: [83, "Can't access a needed shared lib"],

ELIBBAD: [84, "Accessing a corrupted shared lib"],

ELIBSCN: [85, ".lib section in a.out corrupted"],

ELIBMAX: [86, "Attempting to link in too many libs"],

ELIBEXEC: [87, "Attempting to exec a shared library"],

ENOSYS: [88, "Function not implemented"],

ENMFILE: [89, "No more files"],

ENOTEMPTY: [90, "Directory not empty"],

ENAMETOOLONG: [91, "File or path name too long"],

ELOOP: [92, "Too many symbolic links"],

EOPNOTSUPP: [95, "Operation not supported on transport endpoint"],

EPFNOSUPPORT: [96, "Protocol family not supported"],

ECONNRESET: [104, "Connection reset by peer"],

ENOBUFS: [105, "No buffer space available"],

EAFNOSUPPORT: [106, "Address family not supported by protocol family"],

EPROTOTYPE: [107, "Protocol wrong type for socket"],

ENOTSOCK: [108, "Socket operation on non-socket"],

ENOPROTOOPT: [109, "Protocol not available"],

ESHUTDOWN: [110, "Can't send after socket shutdown"],

ECONNREFUSED: [111, "Connection refused"],

EADDRINUSE: [112, "Address already in use"],

ECONNABORTED: [113, "Connection aborted"],

ENETUNREACH: [114, "Network is unreachable"],

ENETDOWN: [115, "Network interface is not configured"],

ETIMEDOUT: [116, "Connection timed out"],

EHOSTDOWN: [117, "Host is down"],

EHOSTUNREACH: [118, "Host is unreachable"],

EINPROGRESS: [119, "Connection already in progress"],

EALREADY: [120, "Socket already connected"],

EDESTADDRREQ: [121, "Destination address required"],

EMSGSIZE: [122, "Message too long"],

EPROTONOSUPPORT: [123, "Unknown protocol"],

ESOCKTNOSUPPORT: [124, "Socket type not supported"],

EADDRNOTAVAIL: [125, "Address not available"],

ENETRESET: [126, ""],

EISCONN: [127, "Socket is already connected"],

ENOTCONN: [128, "Socket is not connected"],

ETOOMANYREFS: [129, ""],

EPROCLIM: [130, ""],

EUSERS: [131, ""],

EDQUOT: [132, ""],

ESTALE: [133, ""],

ENOTSUP: [134, "Not supported"],

ENOMEDIUM: [135, "No medium (in tape drive)"],

ENOSHARE: [136, "No such host or network path"],

ECASECLASH: [137, "Filename exists with different case"],

EILSEQ: [138, ""],

EOVERFLOW: [139, "Value too large for defined data type"]

};

for (const k in exports.E)

exports.E[k].push(k);

function l(val, list) {

for (const k in list)

if (val == list[k][0])

return k;

return null;

}

exports.X = {

RANGE: function (p) {

try {

const m = Process.getModuleByAddress(p);

return `${p} (${m != null ? m.name : 'null'})`;

}

catch (e) {

return `${p}`;

}

},

LINKAT: function (f) {

if (f == exports.AT_.AT_SYMLINK_FOLLOW)

return "AT_SYMLINK_FOLLOW";

else

return 0;

},

PRCTL_OPT: function (f) {

return l(f, exports.PR_.OPT);

},

PTRACE: function (f) {

return l(f, exports.PTRACE_);

},

TYPEID: function (f) {

return l(f, exports.K);

},

XATTR: function (f) {

return ["default", "XATTR_CREATE", "XATTR_REPLACE"][f];

},

FNCTL: function (f) {

return l(f, F_);

},

SIG: function (f) {

return l(f, exports.S);

},

MADV: function (f) {

return l(f, exports.MADV_);

},

MCL: function (f) {

return l(f, exports.MCL_);

},

MAP: function (f) {

return stringifyBitmap(f, exports.MAP_);

},

MS: function (f) {

return l(f, exports.MS_);

},

ERR: function (f) {

for (const k in exports.E)

if (f == exports.E[k][0])

return k + "/*" + exports.E[k][1] + "*/";

return null;

},

ATTR: function (f) {

return f;

},

O_FLAG: function (f) {

return stringifyBitmap(f, exports.O_);

},

O_MODE: function (f) {

return stringifyBitmap(f, exports.O_);

},

UMOUNT: function (f) {

return stringifyBitmap(f, exports.MNT_);

},

MFD: function (f) {

return stringifyBitmap(f, exports.MFD);

},

MPROT: function (f) {

if (f == PROT_NONE)

return "PROT_NONE";

return stringifyBitmap(f, exports.PROT_);

},

};

},{}],3:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.LinuxArm64InterruptorAgent = void 0;

const InterruptorAgent_1 = require("../common/InterruptorAgent");

const InterruptorException_1 = require("../common/InterruptorException");

const DataTypes_1 = require("../common/DataTypes");

const DataLabels_1 = require("../common/DataLabels");

const LinuxArm64Flags_1 = require("./LinuxArm64Flags");

const SVC_NUM = 0;

const SVC_NAME = 1;

const SVC_ARG = 3;

const SVC_RET = 4;

const SVC_ERR = 5;

const A = {

DFD: { t: DataTypes_1.T.INT32, n: "dfd", l: DataLabels_1.L.DFD },

OLD_DFD: { t: DataTypes_1.T.INT32, n: "old_dfd", l: DataLabels_1.L.DFD },

NEW_DFD: { t: DataTypes_1.T.INT32, n: "new_dfd", l: DataLabels_1.L.DFD },

FD: { t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD },

LFD: { t: DataTypes_1.T.ULONG, n: "fd", l: DataLabels_1.L.FD },

CONST_PATH: { t: DataTypes_1.T.STRING, n: "path", l: DataLabels_1.L.PATH, c: true },

CONST_NAME: { t: DataTypes_1.T.STRING, n: "name", c: true },

OLD_NAME: { t: DataTypes_1.T.CHAR_BUFFER, n: "old_name", c: true },

NEW_NAME: { t: DataTypes_1.T.CHAR_BUFFER, n: "new_name", c: true },

CONST_FNAME: { t: DataTypes_1.T.STRING, n: "filename", c: true },

SIZE: { t: DataTypes_1.T.UINT32, n: "size", l: DataLabels_1.L.SIZE },

LEN: { t: DataTypes_1.T.ULONG, n: "length", l: DataLabels_1.L.SIZE },

SIGNED_LEN: { t: DataTypes_1.T.LONG, n: "length", l: DataLabels_1.L.SIZE },

XATTR: { t: DataTypes_1.T.INT32, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.XATTR },

PID: { t: DataTypes_1.T.INT32, n: "pid", l: DataLabels_1.L.PID },

UID: { t: DataTypes_1.T.UINT32, n: "user", l: DataLabels_1.L.UID },

GID: { t: DataTypes_1.T.UINT32, n: "group", l: DataLabels_1.L.GID },

SIG: { t: DataTypes_1.T.INT32, n: "sig", l: DataLabels_1.L.SIG },

TID: { t: DataTypes_1.T.INT32, n: "thread" },

CALLER_TID: { t: DataTypes_1.T.INT32, n: "caller_tid" },

PTR: { t: DataTypes_1.T.POINTER64, n: "value" },

START_ADDR: { t: DataTypes_1.T.POINTER64, n: "start_addr", l: DataLabels_1.L.VADDR, f: LinuxArm64Flags_1.X.RANGE },

ADDR: { t: DataTypes_1.T.POINTER64, n: "addr", l: DataLabels_1.L.VADDR, f: LinuxArm64Flags_1.X.RANGE },

CONST_PTR: { t: DataTypes_1.T.POINTER64, n: "value", c: true },

MPROT: { t: DataTypes_1.T.INT32, n: "prot", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.MPROT }

};

const RET = {

INFO: { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EAGAIN, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.EPERM] },

STAT: { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ELOOP, LinuxArm64Flags_1.E.ENAMETOOLONG, LinuxArm64Flags_1.E.ENOENT, LinuxArm64Flags_1.E.ENOMEM, LinuxArm64Flags_1.E.ENOTDIR, LinuxArm64Flags_1.E.EOVERFLOW] },

LINK: { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EEXIST, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EIO, LinuxArm64Flags_1.E.ELOOP, LinuxArm64Flags_1.E.EMLINK, LinuxArm64Flags_1.E.ENAMETOOLONG, LinuxArm64Flags_1.E.ENOENT, LinuxArm64Flags_1.E.ENOMEM, LinuxArm64Flags_1.E.ENOSPC, LinuxArm64Flags_1.E.ENOTDIR, LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EROFS, LinuxArm64Flags_1.E.EXDEV] },

OPEN: { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EEXIST, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.ENODEV, LinuxArm64Flags_1.E.ENOENT, LinuxArm64Flags_1.E.ENOMEM, LinuxArm64Flags_1.E.ENOSPC, LinuxArm64Flags_1.E.ENOTDIR, LinuxArm64Flags_1.E.ENXIO, LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EROFS, LinuxArm64Flags_1.E.ETXTBSY, LinuxArm64Flags_1.E.EFBIG, LinuxArm64Flags_1.E.EINTR, LinuxArm64Flags_1.E.EISDIR, LinuxArm64Flags_1.E.ELOOP, LinuxArm64Flags_1.E.ENAMETOOLONG, LinuxArm64Flags_1.E.EMFILE, LinuxArm64Flags_1.E.ENFILE, LinuxArm64Flags_1.E.ENOMEM] },

};

RET.SET_XATTR = { t: DataTypes_1.T.INT32, e: RET.STAT.e.concat([LinuxArm64Flags_1.E.EDQUOT, LinuxArm64Flags_1.E.EEXIST, LinuxArm64Flags_1.E.ENODATA, LinuxArm64Flags_1.E.ENOSPC, LinuxArm64Flags_1.E.ENOTSUP, LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.ERANGE]) };

RET.GET_XATTR = { t: DataTypes_1.T.INT32, e: RET.STAT.e.concat([LinuxArm64Flags_1.E.E2BIG, LinuxArm64Flags_1.E.ENODATA, LinuxArm64Flags_1.E.ENOTSUP, LinuxArm64Flags_1.E.ERANGE]) };

RET.LS_XATTR = { t: DataTypes_1.T.INT32, e: RET.STAT.e.concat([LinuxArm64Flags_1.E.E2BIG, LinuxArm64Flags_1.E.ENOTSUP, LinuxArm64Flags_1.E.ERANGE]) };

RET.RM_XATTR = { t: DataTypes_1.T.INT32, e: RET.STAT.e.concat([LinuxArm64Flags_1.E.ENOTSUP, LinuxArm64Flags_1.E.ERANGE]) };

RET.OPENAT = { t: DataTypes_1.T.INT32, n: 'FD', l: DataLabels_1.L.FD, r: 1, e: RET.OPEN.e.concat([LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.ENOTDIR]) };

RET.LINKAT = { t: DataTypes_1.T.INT32, e: RET.LINK.e.concat([LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.ENOTDIR]) };

const SVC = [

[0, "io_setup", 0x00, ["unsigned nr_reqs", "aio_context_t *ctx"]],

[1, "io_destroy", 0x01, ["aio_context_t ctx"]],

[2, "io_submit", 0x02, ["aio_context_t", "long", "struct iocb * *"]],

[3, "io_cancel", 0x03, ["aio_context_t ctx_id", "struct iocb *iocb", "struct io_event *result"]],

[4, "io_getevents", 0x04, ["aio_context_t ctx_id", "long min_nr", "long nr", "struct io_event *events", "struct __kernel_timespec *timeout"]],

[5, "setxattr", 0x05, [A.CONST_PATH, A.CONST_NAME, A.PTR, A.SIZE, A.XATTR], RET.SET_XATTR],

[6, "lsetxattr", 0x06, [A.CONST_PATH, A.CONST_NAME, A.PTR, A.SIZE, A.XATTR], RET.SET_XATTR],

[7, "fsetxattr", 0x07, [A.FD, A.CONST_NAME, A.CONST_PTR, A.SIZE, A.XATTR], RET.SET_XATTR],

[8, "getxattr", 0x08, [A.CONST_PATH, A.CONST_NAME, A.PTR, A.SIZE], RET.GET_XATTR],

[9, "lgetxattr", 0x09, [A.CONST_PATH, A.CONST_NAME, A.PTR, A.SIZE], RET.GET_XATTR],

[10, "fgetxattr", 0x0a, [A.FD, A.CONST_NAME, A.PTR, A.SIZE], RET.GET_XATTR],

[11, "listxattr", 0x0b, [A.CONST_PATH, { t: DataTypes_1.T.CHAR_BUFFER, n: "list", l: DataLabels_1.L.XATTR_LIST, r: 2 }, A.SIZE], RET.LS_XATTR],

[12, "llistxattr", 0x0c, [A.CONST_PATH, { t: DataTypes_1.T.CHAR_BUFFER, n: "list", l: DataLabels_1.L.XATTR_LIST, r: 2 }, A.SIZE], RET.LS_XATTR],

[13, "flistxattr", 0x0d, [A.FD, { t: DataTypes_1.T.CHAR_BUFFER, n: "list", l: DataLabels_1.L.XATTR_LIST, r: 2 }, A.SIZE], RET.LS_XATTR],

[14, "removexattr", 0x0e, [A.CONST_PATH, A.CONST_NAME], RET.RM_XATTR],

[15, "lremovexattr", 0x0f, [A.CONST_PATH, A.CONST_NAME], RET.RM_XATTR],

[16, "fremovexattr", 0x10, [A.FD, A.CONST_PATH, A.CONST_NAME], RET.RM_XATTR],

[17, "getcwd", 0x11, [{ t: DataTypes_1.T.CHAR_BUFFER, n: "path_buff", l: DataLabels_1.L.PATH }, A.SIZE], { t: DataTypes_1.T.CHAR_BUFFER, n: "path_buff", l: DataLabels_1.L.PATH, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENOENT, LinuxArm64Flags_1.E.ERANGE] }],

[18, "lookup_dcookie", 0x12, [{ t: DataTypes_1.T.ULONG, n: "cookie64" }, { t: DataTypes_1.T.CHAR_BUFFER, n: "buffer", l: DataLabels_1.L.XATTR_LIST, r: 2 }, A.SIZE]],

[19, "eventfd2", 0x13, ["unsigned int count", "int flags"]],

[20, "epoll_create1", 0x14, ["int flags"]],

[21, "epoll_ctl", 0x15, ["int epfd", "int op", A.FD, "struct epoll_event *event"]],

[22, "epoll_pwait", 0x16, ["int epfd", "struct epoll_event *events", "int maxevents", "int timeout", "const sigset_t *sigmask", "size_t sigsetsize"]],

[23, "dup", 0x17, [A.FD], { t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD, e: [LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.EBUSY, LinuxArm64Flags_1.E.EINTR, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.EMFILE] }],

[24, "dup3", 0x18, [{ t: DataTypes_1.T.UINT32, n: "old_fd", l: DataLabels_1.L.FD }, { t: DataTypes_1.T.UINT32, n: "old_fd", l: DataLabels_1.L.FD }, { t: DataTypes_1.T.INT32, n: "flags", l: DataLabels_1.L.FLAG }], { t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD, e: [LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.EBUSY, LinuxArm64Flags_1.E.EINTR, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.EMFILE] }],

[25, "fcntl", 0x19, [A.FD, { t: DataTypes_1.T.UINT32, name: "cmd", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.FNCTL }, "unsigned long arg"]],

[26, "inotify_init1", 0x1a, ["int flags"]],

[27, "inotify_add_watch", 0x1b, [A.FD, A.CONST_PATH, "u32 mask"]],

[28, "inotify_rm_watch", 0x1c, [A.FD, "__s32 wd"]],

[29, "ioctl", 0x1d, [A.FD, "unsigned int cmd", "unsigned long arg"]],

[30, "ioprio_set", 0x1e, ["int which", "int who", "int ioprio"]],

[31, "ioprio_get", 0x1f, ["int which", "int who"]],

[32, "flock", 0x20, [A.FD, "unsigned int cmd"]],

[33, "mknodat", 0x21, [A.DFD, A.CONST_NAME, "umode_t mode", "unsigned dev"]],

[34, "mkdirat", 0x22, [A.DFD, A.CONST_FNAME, "umode_t mode"]],

[35, "unlinkat", 0x23, [A.DFD, A.CONST_FNAME, "int flag"]],

[36, "symlinkat", 0x24, ["const char * oldname", A.NEW_DFD, "const char * newname"]],

[37, "linkat", 0x25, [A.OLD_DFD, { t: DataTypes_1.T.POINTER64, n: "value" }, A.NEW_DFD, { t: DataTypes_1.T.POINTER64, n: "value" }, { t: DataTypes_1.T.UINT32, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.LINKAT }], RET.LINKAT],

[38, "renameat", 0x26, [A.OLD_DFD, "const char * oldname", A.NEW_DFD, "const char * newname"]],

[39, "umount2", 0x27, [A.CONST_PATH, { t: DataTypes_1.T.INT32, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.UMOUNT, c: true }]],

[40, "mount", 0x28, ["char *dev_name", "char *dir_name", "char *type", "unsigned long flags", "void *dat"]],

[41, "pivot_root", 0x29, ["const char *new_root", "const char *put_old"]],

[42, "nfsservctl", 0x2a, ["int cmd", "struct nfsctl_arg *argp", "union nfsctl_res *resp"]],

[43, "statfs", 0x2b, [A.CONST_PATH, "struct statfs *buf"]],

[44, "fstatfs", 0x2c, [A.FD, "struct statfs *buf"]],

[45, "truncate", 0x2d, [A.CONST_PATH, A.SIGNED_LEN]],

[46, "ftruncate", 0x2e, [A.FD, A.LEN], RET.OPEN],

[47, "fallocate", 0x2f, [

A.FD, "int mode", "loff_t offset", "loff_t len"

]],

[48, "faccessat", 0x30, [A.DFD, { t: DataTypes_1.T.STRING, n: "filename", c: true }, "int mode"]],

[49, "chdir", 0x31, [{ t: DataTypes_1.T.CHAR_BUFFER, n: "path", l: DataLabels_1.L.PATH, c: true }]],

[50, "fchdir", 0x32, [A.FD], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EIO, LinuxArm64Flags_1.E.ELOOP, LinuxArm64Flags_1.E.ENAMETOOLONG, LinuxArm64Flags_1.E.ENOENT, LinuxArm64Flags_1.E.ENOMEM, LinuxArm64Flags_1.E.ENOTDIR, LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EBADF] }],

[51, "chroot", 0x33, [{ t: DataTypes_1.T.CHAR_BUFFER, n: "path", l: DataLabels_1.L.PATH, c: true }], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EIO, LinuxArm64Flags_1.E.ELOOP, LinuxArm64Flags_1.E.ENAMETOOLONG, LinuxArm64Flags_1.E.ENOENT, LinuxArm64Flags_1.E.ENOMEM, LinuxArm64Flags_1.E.ENOTDIR, LinuxArm64Flags_1.E.EPERM] }],

[52, "fchmod", 0x34, [A.FD, { t: DataTypes_1.T.USHORT, n: "mode", l: DataLabels_1.L.ATTRMODE, f: LinuxArm64Flags_1.X.ATTR }], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EIO, LinuxArm64Flags_1.E.ELOOP, LinuxArm64Flags_1.E.ENAMETOOLONG, LinuxArm64Flags_1.E.ENOENT, LinuxArm64Flags_1.E.ENOMEM, LinuxArm64Flags_1.E.ENOTDIR, LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.EROFS] }],

[53, "fchmodat", 0x35, [A.DFD, A.CONST_PATH, "umode_t mode"]],

[54, "fchownat", 0x36, [A.DFD, A.CONST_PATH, "uid_t user", A.GID, "int fla"]],

[55, "fchown", 0x37, [A.FD, "uid_t user", A.GID]],

[56, "openat", 0x38, [A.DFD,

A.CONST_FNAME, "int flags", { t: DataTypes_1.T.UINT32, n: "mode", l: DataLabels_1.L.O_FLAGS, f: LinuxArm64Flags_1.X.O_MODE }], RET.OPENAT],

[57, "close", 0x39, [A.FD]],

[58, "vhangup", 0x3a, []],

[59, "pipe2", 0x3b, ["int *fildes", "int flags"]],

[60, "quotactl", 0x3c, ["unsigned int cmd", "const char *special", "qid_t id", "void *addr"]],

[61, "getdents64", 0x3d, [{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "struct linux_dirent64 *dirent", "unsigned int count"]],

[62, "lseek", 0x3e, [A.FD,

{ t: DataTypes_1.T.UINT32, n: "offset" },

{ t: DataTypes_1.T.UINT32, n: "whence", l: DataLabels_1.L.SIZE }]],

[63, "read", 0x3f, [A.FD,

{ t: DataTypes_1.T.POINTER64, n: "buf", l: DataLabels_1.L.OUTPUT_BUFFER },

{ t: DataTypes_1.T.UINT32, n: "count", l: DataLabels_1.L.SIZE }

], { t: DataTypes_1.T.UINT32, r: 1, n: "sz", l: DataLabels_1.L.SIZE }],

[64, "write", 0x40, [A.FD, { t: DataTypes_1.T.CHAR_BUFFER, n: "buf", c: true }, { t: DataTypes_1.T.UINT32, n: "count", l: DataLabels_1.L.SIZE }]],

[65, "readv", 0x41, [A.LFD, "const struct iovec *vec", "unsigned long vlen"]],

[66, "writev", 0x42, [A.LFD, "const struct iovec *vec", "unsigned long vlen"]],

[67, "pread64", 0x43, [A.FD, "char *buf", "size_t count", "loff_t pos"]],

[68, "pwrite64", 0x44, [A.FD, "const char *buf", "size_t count", "loff_t pos"]],

[69, "preadv", 0x45, [A.LFD, "const struct iovec *vec", "unsigned long vlen", "unsigned long pos_l", "unsigned long pos_"]],

[70, "pwritev", 0x46, [A.LFD, "const struct iovec *vec", "unsigned long vlen", "unsigned long pos_l", "unsigned long pos_"]],

[71, "sendfile", 0x47, [{ t: DataTypes_1.T.UINT32, n: "out_fd", l: DataLabels_1.L.FD }, { t: DataTypes_1.T.UINT32, n: "in_fd", l: DataLabels_1.L.FD }, "off_t *offset", "size_t count"]],

[72, "pselect6", 0x48, ["int", "fd_set *", "fd_set *", "fd_set *", "struct __kernel_timespec *", "void *["]],

[73, "ppoll", 0x49, ["struct pollfd *", "unsigned int", "struct __kernel_timespec *", "const sigset_t *", "size_"]],

[74, "signalfd4", 0x4a, ["int ufd", "sigset_t *user_mask", "size_t sizemask", "int flags"]],

[75, "vmsplice", 0x4b, [A.FD, "const struct iovec *iov", "unsigned long nr_segs", "unsigned int flags"]],

[76, "splice", 0x4c, [

{ t: DataTypes_1.T.UINT32, n: "fd_in", l: DataLabels_1.L.FD }, "loff_t *off_in", { t: DataTypes_1.T.UINT32, n: "fd_out", l: DataLabels_1.L.FD }, "loff_t *off_out", "size_t len", "unsigned int flags["

]],

[77, "tee", 0x4d, [

{ t: DataTypes_1.T.UINT32, n: "fd_in", l: DataLabels_1.L.FD }, { t: DataTypes_1.T.UINT32, n: "fd_out", l: DataLabels_1.L.FD }, "size_t len", "unsigned int flags"

]],

[78, "readlinkat", 0x4e, [

{ t: DataTypes_1.T.INT32, n: "dfd", l: DataLabels_1.L.DFD },

{ t: DataTypes_1.T.STRING, n: "path", l: DataLabels_1.L.PATH, c: true }, "char *buf", "int bufsiz"

]],

[79, "newfstatat", 0x4f, [

{ t: DataTypes_1.T.INT32, n: "dfd", l: DataLabels_1.L.DFD },

{ t: DataTypes_1.T.STRING, n: "filename", c: true }, "struct stat *statbuf", "int flag"

]],

[80, "fstat", 0x50, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "struct __old_kernel_stat *statbuf"

]],

[81, "sync", 0x51, []],

[82, "fsync", 0x52, [A.FD]],

[83, "fdatasync", 0x53, [A.FD]],

[84, "sync_file_range", 0x54, [A.FD, "loff_t offset", "loff_t nbytes", "unsigned int flags"]],

[85, "timerfd_create", 0x55, ["int clockid", "int flags"]],

[86, "timerfd_settime", 0x56, ["int ufd", "int flags", "const struct __kernel_itimerspec *utmr", "struct __kernel_itimerspec *otmr"]],

[87, "timerfd_gettime", 0x57, ["int ufd", "struct __kernel_itimerspec *otmr"]],

[88, "utimensat", 0x58, [A.DFD, { t: DataTypes_1.T.STRING, n: "filename", c: true }, "struct __kernel_timespec *utimes", "int flags"]],

[89, "acct", 0x59, [

{ t: DataTypes_1.T.STRING, n: "name", c: true }

]],

[90, "capget", 0x5a, ["cap_user_header_t header", "cap_user_data_t dataptr"]],

[91, "capset", 0x5b, ["cap_user_header_t header", "const cap_user_data_t data"]],

[92, "personality", 0x5c, ["unsigned int personality"]],

[93, "exit", 0x5d, [{ t: DataTypes_1.T.INT32, n: "status" }]],

[94, "exit_group", 0x5e, [{ t: DataTypes_1.T.INT32, n: "status" }]],

[95, "waitid", 0x5f, [{ t: DataTypes_1.T.INT32, n: "type_id", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.TYPEID }, { t: DataTypes_1.T.UINT32, n: "id" }, "struct siginfo *infop", "int options", "struct rusage *r"]],

[96, "set_tid_address", 0x60, [{ t: DataTypes_1.T.POINTER32, n: "*tidptr" }], A.CALLER_TID],

[97, "unshare", 0x61, ["unsigned long unshare_flags"]],

[98, "futex", 0x62, ["u32 *uaddr", "int op", "u32 val", "struct __kernel_timespec *utime", "u32 *uaddr2", "u32 val3["]],

[99, "set_robust_list", 0x63, ["struct robust_list_head *head", "size_t len"]],

[100, "get_robust_list", 0x64, ["int pid", "struct robust_list_head * *head_ptr", "size_t *len_ptr"]],

[101, "nanosleep", 0x65, ["struct __kernel_timespec *rqtp", "struct __kernel_timespec *rmtp"]],

[102, "getitimer", 0x66, ["int which", "struct itimerval *value"]],

[103, "setitimer", 0x67, ["int which", "struct itimerval *value", "struct itimerval *ovalue"]],

[104, "kexec_load", 0x68, ["unsigned long entry", "unsigned long nr_segments", "struct kexec_segment *segments", "unsigned long flags"]],

[105, "init_module", 0x69, ["void *umod", "unsigned long len", "const char *uargs"]],

[106, "delete_module", 0x6a, ["const char *name_user", "unsigned int flags"]],

[107, "timer_create", 0x6b, ["clockid_t which_clock", "struct sigevent *timer_event_spec", "timer_t * created_timer_id"]],

[108, "timer_gettime", 0x6c, ["timer_t timer_id", "struct __kernel_itimerspec *setting"]],

[109, "timer_getoverrun", 0x6d, ["timer_t timer_id"]],

[110, "timer_settime", 0x6e, ["timer_t timer_id", "int flags", "const struct __kernel_itimerspec *new_setting", "struct __kernel_itimerspec *old_setting"]],

[111, "timer_delete", 0x6f, ["timer_t timer_id"]],

[112, "clock_settime", 0x70, ["clockid_t which_clock", "const struct __kernel_timespec *tp"]],

[113, "clock_gettime", 0x71, ["clockid_t which_clock", "struct __kernel_timespec *tp"]],

[114, "clock_getres", 0x72, ["clockid_t which_clock", "struct __kernel_timespec *tp"]],

[115, "clock_nanosleep", 0x73, ["clockid_t which_clock", "int flags", "const struct __kernel_timespec *rqtp", "struct __kernel_timespec *rmtp"]],

[116, "syslog", 0x74, ["int type", "char *buf", "int len"]],

[117, "ptrace", 0x75, [{ t: DataTypes_1.T.LONG, n: "request", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.PTRACE }, { t: DataTypes_1.T.LONG, n: "pid", l: DataLabels_1.L.PID }, A.ADDR, "unsigned long data"]],

[118, "sched_setparam", 0x76, [A.PID, "struct sched_param *param"]],

[119, "sched_setscheduler", 0x77, [A.PID, "int policy", "struct sched_param *param"]],

[120, "sched_getscheduler", 0x78, [A.PID]],

[121, "sched_getparam", 0x79, [A.PID, "struct sched_param *param"]],

[122, "sched_setaffinity", 0x7a, [A.PID, "unsigned int len", "unsigned long *user_mask_ptr"]],

[123, "sched_getaffinity", 0x7b, [A.PID, "unsigned int len", "unsigned long *user_mask_ptr"]],

[124, "sched_yield", 0x7c, []],

[125, "sched_get_priority_max", 0x7d, ["int policy"]],

[126, "sched_get_priority_min", 0x7e, ["int policy"]],

[127, "sched_rr_get_interval", 0x7f, [A.PID, "struct __kernel_timespec *interval"]],

[128, "restart_syscall", 0x80, []],

[129, "kill", 0x81, [A.PID, A.SIG]],

[130, "tkill", 0x82, [A.PID, A.SIG]],

[131, "tgkill", 0x83, [{ t: DataTypes_1.T.INT32, n: "thread_grp", l: DataLabels_1.L.PID }, A.PID, A.SIG]],

[132, "sigaltstack", 0x84, ["const struct sigaltstack *uss", "struct sigaltstack *uoss"]],

[133, "rt_sigsuspend", 0x85, ["sigset_t *unewset", "size_t sigsetsize"]],

[134, "rt_sigaction", 0x86, ["int", "const struct sigaction *", "struct sigaction *", "size_t"]],

[135, "rt_sigprocmask", 0x87, ["int how", "sigset_t *set", "sigset_t *oset", "size_t sigsetsize"]],

[136, "rt_sigpending", 0x88, ["sigset_t *set", "size_t sigsetsize"]],

[137, "rt_sigtimedwait", 0x89, ["const sigset_t *uthese", "siginfo_t *uinfo", "const struct __kernel_timespec *uts", "size_t sigsetsize"]],

[138, "rt_sigqueueinfo", 0x8a, [A.PID, A.SIG, "siginfo_t *uinfo"]],

[139, "rt_sigreturn", 0x8b, []],

[140, "setpriority", 0x8c, ["int which", "int who", "int niceval"]],

[141, "getpriority", 0x8d, ["int which", "int who"]],

[142, "reboot", 0x8e, ["int magic1", "int magic2", "unsigned int cmd", "void *arg"]],

[143, "setregid", 0x8f, ["gid_t rgid", "gid_t egid"]],

[144, "setgid", 0x90, [A.GID], RET.INFO],

[145, "setreuid", 0x91, [{ t: DataTypes_1.T.UINT32, n: "real_user", l: DataLabels_1.L.UID }, { t: DataTypes_1.T.UINT32, n: "effective_user", l: DataLabels_1.L.UID }], RET.INFO],

[146, "setuid", 0x92, [A.UID], RET.INFO],

[147, "setresuid", 0x93, [{ t: DataTypes_1.T.UINT32, n: "real_user", l: DataLabels_1.L.UID }, { t: DataTypes_1.T.UINT32, n: "effective_user", l: DataLabels_1.L.UID }, { t: DataTypes_1.T.UINT32, n: "suid", l: DataLabels_1.L.UID }], RET.INFO],

[148, "getresuid", 0x94, [{ t: DataTypes_1.T.POINTER64, n: "real_user", l: DataLabels_1.L.UID }, { t: DataTypes_1.T.POINTER64, n: "effective_user", l: DataLabels_1.L.UID }, { t: DataTypes_1.T.POINTER64, n: "suid", l: DataLabels_1.L.UID }]],

[149, "setresgid", 0x95, [{ t: DataTypes_1.T.UINT32, n: "real_grp", l: DataLabels_1.L.GID }, { t: DataTypes_1.T.UINT32, n: "effective_grp", l: DataLabels_1.L.GID }, { t: DataTypes_1.T.UINT32, n: "sgid", l: DataLabels_1.L.GID }], RET.INFO],

[150, "getresgid", 0x96, [{ t: DataTypes_1.T.POINTER64, n: "real_grp", l: DataLabels_1.L.UID }, { t: DataTypes_1.T.POINTER64, n: "effective_grp", l: DataLabels_1.L.UID }, { t: DataTypes_1.T.POINTER64, n: "sgid", l: DataLabels_1.L.UID }], RET.INFO],

[151, "setfsuid", 0x97, [A.UID], RET.INFO],

[152, "setfsgid", 0x98, [A.GID], RET.INFO],

[153, "times", 0x99, ["struct tms *tbuf"]],

[154, "setpgid", 0x9a, [A.PID, { t: DataTypes_1.T.INT32, n: "pgid", l: DataLabels_1.L.PID }], RET.INFO],

[155, "getpgid", 0x9b, [A.PID]],

[156, "getsid", 0x9c, [A.PID]],

[157, "setsid", 0x9d, [], , RET.INFO],

[158, "getgroups", 0x9e, [A.SIZE, { t: DataTypes_1.T.POINTER64, n: "grouplist", l: DataLabels_1.L.GID }]],

[159, "setgroups", 0x9f, [A.SIZE, { t: DataTypes_1.T.POINTER64, n: "grouplist", l: DataLabels_1.L.GID }], RET.INFO],

[160, "uname", 0xa0, [{ t: DataTypes_1.T.POINTER64, n: "*utsname" }]],

[161, "sethostname", 0xa1, [{ t: DataTypes_1.T.CHAR_BUFFER, n: "name" }, { t: DataTypes_1.T.UINT32, n: "length" }]],

[162, "setdomainname", 0xa2, [{ t: DataTypes_1.T.CHAR_BUFFER, n: "name" }, { t: DataTypes_1.T.UINT32, n: "length" }]],

[163, "getrlimit", 0xa3, ["unsigned int resource", "struct rlimit *rlim"]],

[164, "setrlimit", 0xa4, ["unsigned int resource", "struct rlimit *rlim"]],

[165, "getrusage", 0xa5, ["int who", "struct rusage *ru"]],

[166, "umask", 0xa6, [{ t: DataTypes_1.T.UINT32, n: "mask", l: DataLabels_1.L.ATTRMODE, f: LinuxArm64Flags_1.X.ATTR }]],

[167, "prctl", 0xa7, [{ t: DataTypes_1.T.INT32, n: "opt", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.PRCTL_OPT }, "unsigned long arg2", "unsigned long arg3", "unsigned long arg4", "unsigned long arg5"]],

[168, "getcpu", 0xa8, ["unsigned *cpu", "unsigned *node", "struct getcpu_cache *cache"]],

[169, "gettimeofday", 0xa9, ["struct timeval *tv", "struct timezone *tz"]],

[170, "settimeofday", 0xaa, ["struct timeval *tv", "struct timezone *tz"]],

[171, "adjtimex", 0xab, ["struct __kernel_timex *txc_p"]],

[172, "getpid", 0xac, [], A.PID],

[173, "getppid", 0xad, [], A.PID],

[174, "getuid", 0xae, [], A.UID],

[175, "geteuid", 0xaf, [], A.UID],

[176, "getgid", 0xb0, [], A.GID],

[177, "getegid", 0xb1, [], A.GID],

[178, "gettid", 0xb2, []],

[179, "sysinfo", 0xb3, ["struct sysinfo *info"]],

[180, "mq_open", 0xb4, [

{ t: DataTypes_1.T.STRING, n: "name", c: true }, "int oflag", "umode_t mode", "struct mq_attr *attr"

]],

[181, "mq_unlink", 0xb5, [

{ t: DataTypes_1.T.STRING, n: "name", c: true }

]],

[182, "mq_timedsend", 0xb6, ["mqd_t mqdes", "const char *msg_ptr", "size_t msg_len", "unsigned int msg_prio", "const struct __kernel_timespec *abs_timeout"]],

[183, "mq_timedreceive", 0xb7, ["mqd_t mqdes", "char *msg_ptr", "size_t msg_len", "unsigned int *msg_prio", "const struct __kernel_timespec *abs_timeout"]],

[184, "mq_notify", 0xb8, ["mqd_t mqdes", "const struct sigevent *notification"]],

[185, "mq_getsetattr", 0xb9, ["mqd_t mqdes", "const struct mq_attr *mqstat", "struct mq_attr *omqstat"]],

[186, "msgget", 0xba, ["key_t key", "int msgflg"]],

[187, "msgctl", 0xbb, ["int msqid", "int cmd", "struct msqid_ds *buf"]],

[188, "msgrcv", 0xbc, ["int msqid", "struct msgbuf *msgp", "size_t msgsz", "long msgtyp", "int msgflg"]],

[189, "msgsnd", 0xbd, ["int msqid", "struct msgbuf *msgp", "size_t msgsz", "int msgflg"]],

[190, "semget", 0xbe, ["key_t key", "int nsems", "int semflg"]],

[191, "semctl", 0xbf, ["int semid", "int semnum", "int cmd", "unsigned long arg"]],

[192, "semtimedop", 0xc0, ["int semid", "struct sembuf *sops", "unsigned nsops", "const struct __kernel_timespec *timeout"]],

[193, "semop", 0xc1, ["int semid", "struct sembuf *sops", "unsigned nsops"]],

[194, "shmget", 0xc2, ["key_t key", "size_t size", "int flag"]],

[195, "shmctl", 0xc3, ["int shmid", "int cmd", "struct shmid_ds *buf"]],

[196, "shmat", 0xc4, ["int shmid", "char *shmaddr", "int shmflg"]],

[197, "shmdt", 0xc5, ["char *shmaddr"]],

[198, "socket", 0xc6, ["int", "int", "int"]],

[199, "socketpair", 0xc7, ["int", "int", "int", "int *"]],

[200, "bind", 0xc8, ["int", "struct sockaddr *", "int"]],

[201, "listen", 0xc9, ["int", "int"]],

[202, "accept", 0xca, ["int", "struct sockaddr *", "int *"]],

[203, "connect", 0xcb, ["int", "struct sockaddr *", "int"]],

[204, "getsockname", 0xcc, ["int", "struct sockaddr *", "int *"]],

[205, "getpeername", 0xcd, ["int", "struct sockaddr *", "int *"]],

[206, "sendto", 0xce, ["int", "void *", "size_t", "unsigned", "struct sockaddr *", "int"]],

[207, "recvfrom", 0xcf, ["int", "void *", "size_t", "unsigned", "struct sockaddr *", "int *"]],

[208, "setsockopt", 0xd0, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "int level", "int optname", "char *optval", "int optlen"

]],

[209, "getsockopt", 0xd1, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "int level", "int optname", "char *optval", "int *optlen"

]],

[210, "shutdown", 0xd2, ["int", "int"]],

[211, "sendmsg", 0xd3, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "struct user_msghdr *msg", "unsigned flags"

]],

[212, "recvmsg", 0xd4, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "struct user_msghdr *msg", "unsigned flags"

]],

[213, "readahead", 0xd5, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "loff_t offset", "size_t count"

]],

[214, "brk", 0xd6, ["unsigned long brk"]],

[215, "munmap", 0xd7, [A.ADDR, A.SIZE], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EINVAL] }],

[216, "mremap", 0xd8, [A.ADDR, "unsigned long old_len", "unsigned long new_len", "unsigned long flags", A.ADDR]],

[217, "add_key", 0xd9, ["const char *_type", "const char *_description", "const void *_payload", "size_t plen", "key_serial_t destringid"]],

[218, "request_key", 0xda, ["const char *_type", "const char *_description", "const char *_callout_info", "key_serial_t destringid"]],

[219, "keyctl", 0xdb, ["int cmd", "unsigned long arg2", "unsigned long arg3", "unsigned long arg4", "unsigned long arg5"]],

[220, "clone", 0xdc, ["unsigned long", "unsigned long", "int *", "int *", "unsigned long"]],

[221, "execve", 0xdd, [

{ t: DataTypes_1.T.STRING, n: "filename", c: true }, "const char *const *argv", "const char *const *envp"

]],

[222, "mmap", 0xde, [A.START_ADDR, A.SIZE, A.MPROT, { t: DataTypes_1.T.INT32, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.MAP },

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.MFD }, { t: DataTypes_1.T.UINT32, n: "offset", l: DataLabels_1.L.SIZE }], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EAGAIN, LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENFILE, LinuxArm64Flags_1.E.ENODEV, LinuxArm64Flags_1.E.ENOMEM, LinuxArm64Flags_1.E.ETXTBSY] }],

[223, "fadvise64", 0xdf, [{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "loff_t offset", A.SIZE, "int advice"]],

[224, "swapon", 0xe0, ["const char *specialfile", "int swap_flags"]],

[225, "swapoff", 0xe1, ["const char *specialfile"]],

[226, "mprotect", 0xe2, [A.ADDR, A.SIZE, A.MPROT], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EACCES, LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENOMEM] }],

[227, "msync", 0xe3, [A.ADDR, A.SIZE, { t: DataTypes_1.T.ULONG, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.MS }], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EBUSY, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENOMEM] }],

[228, "mlock", 0xe4, [A.ADDR, A.SIZE], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENOMEM] }],

[229, "munlock", 0xe5, [A.ADDR, A.SIZE], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENOMEM] }],

[230, "mlockall", 0xe6, [{ t: DataTypes_1.T.INT32, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.MCL }], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENOMEM] }],

[231, "munlockall", 0xe7, [], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EPERM, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.ENOMEM] }],

[232, "mincore", 0xe8, [A.ADDR, A.SIZE, "unsigned char * vec"]],

[233, "madvise", 0xe9, [A.ADDR, A.SIG, { t: DataTypes_1.T.INT32, n: "behavior", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.MADV }], { t: DataTypes_1.T.INT32, e: [LinuxArm64Flags_1.E.EAGAIN, LinuxArm64Flags_1.E.EBADF, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.EIO, LinuxArm64Flags_1.E.ENOMEM] }],

[234, "remap_file_pages", 0xea, ["unsigned long start", "unsigned long size", "unsigned long prot", "unsigned long pgoff", "unsigned long flags"]],

[235, "mbind", 0xeb, [A.ADDR, A.LEN, "unsigned long mode", "const unsigned long *nmask", "unsigned long maxnode", "unsigned flags"]],

[236, "get_mempolicy", 0xec, ["int *policy", "unsigned long *nmask", "unsigned long maxnode", "unsigned long addr", "unsigned long flags"]],

[237, "set_mempolicy", 0xed, ["int mode", "const unsigned long *nmask", "unsigned long maxnode"]],

[238, "migrate_pages", 0xee, [{ t: DataTypes_1.T.INT32, n: "pid", l: DataLabels_1.L.PID }, "unsigned long maxnode", "const unsigned long *from", "const unsigned long *to"]],

[239, "move_pages", 0xef, [{ t: DataTypes_1.T.INT32, n: "pid", l: DataLabels_1.L.PID }, "unsigned long nr_pages", "const void * *pages", "const int *nodes", "int *status", "int flags"]],

[240, "rt_tgsigqueueinfo", 0xf0, [{ t: DataTypes_1.T.INT32, n: "tgid", l: DataLabels_1.L.PID }, A.PID, A.SIG, "siginfo_t *uinfo"]],

[241, "perf_event_open", 0xf1, ["struct perf_event_attr *attr_uptr", { t: DataTypes_1.T.INT32, n: "pid", l: DataLabels_1.L.PID }, "int cpu", "int group_fd", "unsigned long flags"]],

[242, "accept4", 0xf2, ["int", "struct sockaddr *", "int *", "int"]],

[243, "recvmmsg", 0xf3, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "struct mmsghdr *msg", "unsigned int vlen", "unsigned flags", "struct __kernel_timespec *timeout"

]],

[244, "not implemented", 0xf4, []],

[245, "not implemented", 0xf5, []],

[246, "not implemented", 0xf6, []],

[247, "not implemented", 0xf7, []],

[248, "not implemented", 0xf8, []],

[249, "not implemented", 0xf9, []],

[250, "not implemented", 0xfa, []],

[251, "not implemented", 0xfb, []],

[252, "not implemented", 0xfc, []],

[253, "not implemented", 0xfd, []],

[254, "not implemented", 0xfe, []],

[255, "not implemented", 0xff, []],

[256, "not implemented", 0x100, []],

[257, "not implemented", 0x101, []],

[258, "not implemented", 0x102, []],

[259, "not implemented", 0x103, []],

[260, "wait4", 0x104, [A.PID, "int *stat_addr", "int options", "struct rusage *ru"]],

[261, "prlimit64", 0x105, [A.PID, "unsigned int resource", "const struct rlimit64 *new_rlim", "struct rlimit64 *old_rlim"]],

[262, "fanotify_init", 0x106, ["unsigned int flags", "unsigned int event_f_flags"]],

[263, "fanotify_mark", 0x107, ["int fanotify_fd", "unsigned int flags", "u64 mask", { t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "const char *pathname"]],

[264, "name_to_handle_at", 0x108, [{ t: DataTypes_1.T.INT32, n: "dfd", l: DataLabels_1.L.DFD }, { t: DataTypes_1.T.STRING, n: "name", c: true }, "struct file_handle *handle", "int *mnt_id", "int flag"]],

[265, "open_by_handle_at", 0x109, ["int mountdirfd", "struct file_handle *handle", "int flags"]],

[266, "clock_adjtime", 0x10a, ["clockid_t which_clock", "struct __kernel_timex *tx"]],

[267, "syncfs", 0x10b, [{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }]],

[268, "setns", 0x10c, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "int nstype"

]],

[269, "sendmmsg", 0x10d, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "struct mmsghdr *msg", "unsigned int vlen", "unsigned flags"

]],

[270, "process_vm_readv", 0x10e, [{ t: DataTypes_1.T.INT32, n: "pid", l: DataLabels_1.L.PID }, "const struct iovec *lvec", "unsigned long liovcnt", "const struct iovec *rvec", "unsigned long riovcnt", "unsigned long flags"]],

[271, "process_vm_writev", 0x10f, [{ t: DataTypes_1.T.INT32, n: "pid", l: DataLabels_1.L.PID }, "const struct iovec *lvec", "unsigned long liovcnt", "const struct iovec *rvec", "unsigned long riovcnt", "unsigned long flags"]],

[272, "kcmp", 0x110, [{ t: DataTypes_1.T.INT32, n: "pid1", l: DataLabels_1.L.PID }, { t: DataTypes_1.T.INT32, n: "pid2", l: DataLabels_1.L.PID }, "int type", "unsigned long idx1", "unsigned long idx2"]],

[273, "finit_module", 0x111, [

{ t: DataTypes_1.T.UINT32, n: "fd", l: DataLabels_1.L.FD }, "const char *uargs", "int flags"

]],

[274, "sched_setattr", 0x112, [A.PID, "struct sched_attr *attr", "unsigned int flags"]],

[275, "sched_getattr", 0x113, [A.PID, "struct sched_attr *attr", "unsigned int size", "unsigned int flags"]],

[276, "renameat2", 0x114, [{ t: DataTypes_1.T.INT32, n: "old_dfd", l: DataLabels_1.L.DFD }, "const char *oldname", { t: DataTypes_1.T.INT32, n: "new_dfd", l: DataLabels_1.L.DFD }, "const char *newname", "unsigned int flags"]],

[277, "seccomp", 0x115, ["unsigned int op", "unsigned int flags", "void *uargs"]],

[278, "getrandom", 0x116, [{ t: DataTypes_1.T.CHAR_BUFFER, n: "buf", l: DataLabels_1.L.OUTPUT_BUFFER }, "size_t count", "unsigned int flags"]],

[279, "memfd_create", 0x117, [{ t: DataTypes_1.T.CHAR_BUFFER, n: "filename", l: DataLabels_1.L.PATH }, { t: DataTypes_1.T.UINT32, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.MFD }], { t: DataTypes_1.T.UINT32, n: "mfd", l: DataLabels_1.L.FD, e: [LinuxArm64Flags_1.E.EFAULT, LinuxArm64Flags_1.E.EINVAL, LinuxArm64Flags_1.E.EMFILE, LinuxArm64Flags_1.E.ENFILE, LinuxArm64Flags_1.E.ENOMEM] }],

[280, "bpf", 0x118, ["int cmd", "union bpf_attr *attr", "unsigned int size"]],

[281, "execveat", 0x119, [{ t: DataTypes_1.T.INT32, n: "dfd", l: DataLabels_1.L.DFD }, { t: DataTypes_1.T.STRING, n: "filename", c: true }, "const char *const *argv", "const char *const *envp", "int flags"]],

[282, "userfaultfd", 0x11a, [

{ t: DataTypes_1.T.UINT32, n: "flags", l: DataLabels_1.L.FLAG, f: LinuxArm64Flags_1.X.O_MODE }

]],

[283, "membarrier", 0x11b, ["int cmd", "int flags"]],

[284, "mlock2", 0x11c, ["unsigned long start", A.SIZE, "int flags"]],

[285, "copy_file_range", 0x11d, [

{ t: DataTypes_1.T.UINT32, n: "fd_in", l: DataLabels_1.L.FD }, "loff_t *off_in", { t: DataTypes_1.T.UINT32, n: "fd_out", l: DataLabels_1.L.FD }, "loff_t *off_out", A.SIZE, "unsigned int flags"

]],

[286, "preadv2", 0x11e, [A.LFD, "const struct iovec *vec", "unsigned long vlen", "unsigned long pos_l", "unsigned long pos_h", "rwf_t flags"]],

[287, "pwritev2", 0x11f, [A.LFD, "const struct iovec *vec", "unsigned long vlen", "unsigned long pos_l", "unsigned long pos_h", "rwf_t flags"]],

[288, "pkey_mprotect", 0x120, [A.ADDR, A.SIZE, "unsigned long prot", "int pkey"]],

[289, "pkey_alloc", 0x121, ["unsigned long flags", "unsigned long init_val"]],

[290, "pkey_free", 0x122, ["int pkey"]],

[291, "statx", 0x123, [A.DFD, A.CONST_PATH, "unsigned flags", "unsigned mask", "struct statx *buffer"]]

];

const SVC_MAP_NUM = {};

const SVC_MAP_NAME = {};

SVC.map(x => {

SVC_MAP_NAME[x[1]] = x;

SVC_MAP_NUM[x[0]] = x;

});

class LinuxArm64InterruptorAgent extends InterruptorAgent_1.InterruptorAgent {

constructor(pConfig) {

super(pConfig);

this.filter_name = [];

this.filter_num = [];

this.svc_hk = {};

this.hvc_hk = {};

this.smc_hk = {};

this.irq_hk = {};

this.configure(pConfig);

}

configure(pConfig) {

if (pConfig == null)

return;

for (let k in pConfig) {

switch (k) {

case 'svc':

for (let s in pConfig.svc)

this.onSupervisorCall(s, pConfig.svc[s]);

break;

case 'hvc':

for (let s in pConfig.hvc)

this.onHypervisorCall(s.parseInt(16), pConfig.svc[s]);

break;

case 'filter_name':

this.filter_name = pConfig.filter_name;

break;

case 'filter_num':

this.filter_num = pConfig.filter_num;

break;

}

}

this.exclude.svc = pConfig.exclude.hasOwnProperty("svc") ? pConfig.exclude.svc : [];

this.exclude.hvc = pConfig.exclude.hasOwnProperty("hvc") ? pConfig.exclude.hvc : [];

this.exclude.smc = pConfig.exclude.hasOwnProperty("smc") ? pConfig.exclude.smc : [];

this.prepareExcludedSyscalls(this.exclude.syscalls);

this.setupBuiltinHook();

}

prepareExcludedSyscalls(pSyscalls) {

pSyscalls.map(svcName => {

this.exclude.svc.push(SVC_MAP_NAME[svcName][0]);

});

}

onSupervisorCall(pIntName, pHooks) {

const sc = SVC_MAP_NAME[pIntName];

if (sc == null)

throw InterruptorException_1.InterruptorGenericException.UNKNOW_SYSCALL(pIntName);

if (pHooks.hasOwnProperty('onEnter') || pHooks.hasOwnProperty('onLeave')) {

this.svc_hk[sc[0]] = pHooks;

console.log("[SVC HOOK]" + pIntName + "(int=" + sc[0] + ")");

}

}

onHypervisorCall(pIntNum, pHooks) {

if (pHooks.hasOwnProperty('onEnter') || pHooks.hasOwnProperty('onLeave')) {

this.hvc_hk[pIntNum] = pHooks;

}

}

setupBuiltinHook() {

}

locatePC(pContext) {

let l = "";

const r = Process.findRangeByAddress(pContext.pc);

if (this.output.tid)

l += `[TID=${Process.getCurrentThreadId()}]`;

if (this.output.module) {

if (r != null) {

l = `[${r.file != null ? r.file.path : '<no_path>'} +${pContext.pc.sub(r.base)}]`;

;

}

else {

l = `[<unknow> lr=${pContext.lr}]`;

}

}

if (this.output.lr)

l += `[lr=${pContext.lr}]`;

return l;

}

startOnLoad(pModuleRegExp, pCondition = null) {

let self = this, do_dlopen = null, call_ctor = null, match = null;

Process.findModuleByName('linker64').enumerateSymbols().forEach(sym => {

if (sym.name.indexOf('do_dlopen') >= 0) {

do_dlopen = sym.address;

}

else if (sym.name.indexOf('call_constructor') >= 0) {

call_ctor = sym.address;

}

});

Interceptor.attach(do_dlopen, function (args) {

const p = args[0].readUtf8String();

if (p != null && pModuleRegExp.exec(p) != null) {

console.log(p);

match = p;

}

});

Interceptor.attach(call_ctor, {

onEnter: function () {

if (match == null)

return;

if (pCondition !== null) {

if (!(pCondition)(match, this)) {

match = null;

return;

}

}

console.warn("[INTERRUPTOR][STARTING] Module'" + match + "'is loading, tracer will start");

match = null;

self.start();

}

});

}

traceSyscall(pContext, pHookCfg = null) {

if (this.exclude.svc.indexOf(pContext.x8.toInt32()) > -1)

return;

const sys = SVC_MAP_NUM[pContext.x8.toInt32()];

var inst = "SVC";

if (sys == null) {

console.log('[' + this.locatePC(pContext.pc) + '] \x1b[35;01m' + inst + '(' + pContext.x8 + ')\x1b[0m Syscall=<unknow>');

return;

}

pContext.dxcRET = sys[SVC_RET];

let s = "", p = "", t = null;

pContext.dxcOpts = [];

sys[3].map((vVal, vOff) => {

const rVal = pContext["x" + vOff];

if (typeof vVal === "string") {

p += ` ${vVal} = ${rVal} ,`;

}

else {

p += ` ${vVal.n} = `;

switch (vVal.l) {

case DataLabels_1.L.DFD:

t = rVal.toInt32();

if (t >= 0)

p += `${t} `;

else if (t == LinuxArm64Flags_1.AT_.AT_FDCWD)

p += "AT_FDCWD";

else

p += rVal + "ERR?";

break;

case DataLabels_1.L.MFD:

t = rVal.toInt32();

if (t >= 0)

p += `${t} ${pContext.dxcFD[rVal.toInt32() + ""]} `;

else if ((t & LinuxArm64Flags_1.MAP_.MAP_ANONYMOUS[0]) == LinuxArm64Flags_1.MAP_.MAP_ANONYMOUS[0])

p += `${t} IGNORED `;

else

p += t + " ";

return;

case DataLabels_1.L.FD:

t = rVal.toInt32();

if (t >= 0)

p += `${t} ${pContext.dxcFD[t + ""]} `;

else if (t == LinuxArm64Flags_1.AT_.AT_FDCWD)

p += "AT_FDCWD";

else

p += rVal + " ";

break;

case DataLabels_1.L.VADDR:

if (vVal.f == null) {

p += pContext.dxcOpts[vOff] = rVal;

break;

}

case DataLabels_1.L.FLAG:

p += `${(vVal.f)(rVal)}`;

pContext.dxcOpts[vOff] = rVal;

break;

default:

switch (vVal.t) {

case DataTypes_1.T.STRING:

p += pContext.dxcOpts[vOff] = rVal.readCString();

break;

case DataTypes_1.T.CHAR_BUFFER:

p += pContext.dxcOpts[vOff] = rVal.readCString();

break;

case DataTypes_1.T.UINT32:

default:

p += pContext.dxcOpts[vOff] = rVal;

break;

}

break;

}

p += ',';

}

});

s = `${sys[1]} ( ${p.slice(0, -1)} ) `;

if (this.output.flavor == InterruptorAgent_1.InterruptorAgent.FLAVOR_DXC) {

pContext.log = this.locatePC(pContext) + ' \x1b[35;01m' + inst + '::' + pContext.x8 + ' \x1b[0m ' + s;

}

}

getSyscallError(pErrRet, pErrEnum) {

for (let i = 0; i < pErrEnum.length; i++) {

if (pErrRet === pErrEnum[i][0]) {

return pErrRet + ' ' + pErrEnum[i][2];

}

}

return pErrRet;

}

traceSyscallRet(pContext, pHookCfg = null) {

if (this.exclude.svc.indexOf(pContext.x8.toInt32()) > -1)

return;

let ret = pContext.dxcRET;

if (ret != null) {

switch (ret.l) {

case DataLabels_1.L.SIZE:

if (this.output.dump_buff)

ret = "(len=" + pContext.x0 + ")" + pContext["x" + ret.r].readCString();

else

ret = pContext.x0;

break;

case DataLabels_1.L.DFD:

case DataLabels_1.L.FD:

if (pContext.x0 >= 0) {

if (pContext.dxcFD == null)

pContext.dxcFD = {};

console.log(ret.r, pContext.dxcOpts[ret.r]);

pContext.dxcFD[pContext.x0.toInt32() + ""] = pContext.dxcOpts[ret.r];

ret = "(" + (DataLabels_1.L.DFD == ret.l ? "D" : "") + "FD)" + pContext.x0;

}

else if (ret.e) {

let err = this.getSyscallError(pContext.x0, ret.e);

ret = "(ERROR)" + err[2] + " " + err[1] + " ";

}

else {

ret = "(ERROR)" + pContext.x0;

}

break;

default:

if (ret.e != null) {

ret = this.getSyscallError(pContext.x0, ret.e);

if (ret == 0) {

ret = pContext.x0 + 'SUCCESS';

}

}

else

ret = pContext.x0;

break;

}

}

else {

ret = pContext.x0;

}

console.log(pContext.log + '>' + ret);

}

trace(pStalkerInterator, pInstruction, pExtra) {

const self = this;

let keep = 1;

if (pExtra.onLeave == 1) {

pStalkerInterator.putCallout(function (context) {

self.traceSyscallRet(context);

const hook = self.svc_hk[context.x8.toInt32()];

if (hook == null)

return;

if (hook.onLeave != null) {

(hook.onLeave)(context);

}

});

pExtra.onLeave = null;

}

if (pInstruction.mnemonic === 'svc') {

pExtra.onLeave = 1;

pStalkerInterator.putCallout(function (context) {

if (context.dxcFD == null)

context.dxcFD = {};

const hook = self.svc_hk[context.x8.toInt32()];

self.traceSyscall(context, hook);

if (hook != null && hook.onEnter != null)

(hook.onEnter)(context);

});

}

return keep;

}

}

exports.LinuxArm64InterruptorAgent = LinuxArm64InterruptorAgent;

},{"../common/DataLabels":6,"../common/DataTypes":7,"../common/InterruptorAgent":8,"../common/InterruptorException":9,"./LinuxArm64Flags":2}],4:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.LinuxArm64InterruptorFactory = void 0;

const AbstractInterruptorFactory_1 = require("../common/AbstractInterruptorFactory");

const LinuxArm64InterruptorAgent_1 = require("./LinuxArm64InterruptorAgent");

class LinuxArm64InterruptorFactory extends AbstractInterruptorFactory_1.AbstractInterruptorFactory {

constructor(pOptions = null) {

super(pOptions);

}

utils() {

return {

toScanPattern: AbstractInterruptorFactory_1.AbstractInterruptorFactory.toScanPattern

};

}

newAgentTracer(pConfig) {

return new LinuxArm64InterruptorAgent_1.LinuxArm64InterruptorAgent(pConfig);

}

newStandaloneTracer() {

return null;

}

}

exports.LinuxArm64InterruptorFactory = LinuxArm64InterruptorFactory;

},{"../common/AbstractInterruptorFactory":5,"./LinuxArm64InterruptorAgent":3}],5:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.AbstractInterruptorFactory = void 0;

class AbstractInterruptorFactory {

constructor(pOptions) {

this.opts = null;

this.opts = pOptions;

}

static toScanPattern(pString) {

return pString.split('').map(c => c = c.charCodeAt(0).toString(16)).join(' ');

}

getOptions() {

return this.opts;

}

}

exports.AbstractInterruptorFactory = AbstractInterruptorFactory;

},{}],6:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.L = void 0;

var L;

(function (L) {

L[L["PATH"] = 0] = "PATH";

L[L["SIZE"] = 1] = "SIZE";

L[L["FD"] = 2] = "FD";

L[L["DFD"] = 3] = "DFD";

L[L["FLAG"] = 4] = "FLAG";

L[L["ATTRMODE"] = 5] = "ATTRMODE";

L[L["O_FLAGS"] = 6] = "O_FLAGS";

L[L["VADDR"] = 7] = "VADDR";

L[L["MPROT"] = 8] = "MPROT";

L[L["OUTPUT_BUFFER"] = 9] = "OUTPUT_BUFFER";

L[L["PID"] = 10] = "PID";

L[L["ERR"] = 11] = "ERR";

L[L["SIG"] = 12] = "SIG";

L[L["XATTR_LIST"] = 13] = "XATTR_LIST";

L[L["F_"] = 14] = "F_";

L[L["MFD"] = 15] = "MFD";

L[L["UID"] = 16] = "UID";

L[L["GID"] = 17] = "GID";

L[L["UTSNAME"] = 18] = "UTSNAME";

})(L = exports.L || (exports.L = {}));

},{}],7:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.T = void 0;

var T;

(function (T) {

T[T["INT32"] = 0] = "INT32";

T[T["UINT32"] = 1] = "UINT32";

T[T["LONG"] = 2] = "LONG";

T[T["ULONG"] = 3] = "ULONG";

T[T["SHORT"] = 4] = "SHORT";

T[T["USHORT"] = 5] = "USHORT";

T[T["FLOAT"] = 6] = "FLOAT";

T[T["DOUBLE"] = 7] = "DOUBLE";

T[T["CHAR"] = 8] = "CHAR";

T[T["STRING"] = 9] = "STRING";

T[T["CHAR_BUFFER"] = 10] = "CHAR_BUFFER";

T[T["POINTER32"] = 11] = "POINTER32";

T[T["POINTER64"] = 12] = "POINTER64";

T[T["STRUCT"] = 13] = "STRUCT";

})(T = exports.T || (exports.T = {}));

},{}],8:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.InterruptorAgent = void 0;

const InterruptorException_1 = require("./InterruptorException");

const Coverage_1 = require("../utilities/Coverage");

let CTR = 0;

class InterruptorAgent {

constructor(pOptions) {

this.uid = 0;

this.ranges = new Map();

this.modules = [];

this.pid = -1;

this.tid = -1;

this.followFork = false;

this.followThread = false;

this.coverage = null;

this.exclude = {

modules: [],

syscalls: []

};

this.moduleFilter = null;

this.output = {

flavor: "dxc",

tid: true,

pid: false,

module: true,

dump_buff: true,

highlight: {

syscalls: []

}

};

this.uid = CTR++;

this.parseOptions(pOptions);

}

parseOptions(pConfig) {

for (let k in pConfig) {

switch (k) {

case 'pid':

if (typeof pConfig.pid !== "number")

throw InterruptorException_1.InterruptorGenericException.INVALID_PID();

this.pid = pConfig.pid;

break;

case 'tid':

if (typeof pConfig.tid !== "number")

throw InterruptorException_1.InterruptorGenericException.INVALID_TID();

this.tid = pConfig.tid;

break;

case 'coverage':

this.coverage = Coverage_1.CoverageAgent.from(pConfig.coverage, this);

break;

case 'followFork':

this.followFork = (typeof pConfig.followFork !== "boolean" ? false : pConfig.followFork);

break;

case 'followThread':

this.followThread = (typeof pConfig.followFork !== "boolean" ? false : pConfig.followFork);

break;

case 'exclude':

for (k in pConfig.exclude) {

this.exclude[k] = pConfig.exclude[k];

}

break;

case 'output':

for (k in pConfig.output) {

this.output[k] = pConfig.output[k];

}

break;

case 'moduleFilter':

this.moduleFilter = pConfig.moduleFilter;

break;

}

}

}

isTrackCoverage() {

return (this.coverage != null && this.coverage.enabled);

}

processBbsCoverage(pStalkerEvents) {

pStalkerEvents.forEach((e) => {

this.coverage.processStalkerEvent(e);

});

}

filterModuleScope() {

let map;

if (this.moduleFilter != null) {

map = new ModuleMap((m) => {

if ((this.moduleFilter)(m)) {

return true;

}

Stalker.exclude(m);

return false;

});

}

else {

map = new ModuleMap((m) => {

if (this.exclude.modules.indexOf(m.name) == -1) {

return true;

}

Stalker.exclude(m);

return false;

});

}

this.modules = map.values();

for (const module of this.modules) {

const ranges = module.enumerateRanges("--x");

this.ranges.set(module.base, ranges);

}

}

trace(pStalkerInterator, pInstruction, pExtra) {

return 1;

}

startOnLoad(pModuleRegExp, pCondition = null) {

return new Error("Dynamic loading is not supported");

}

start() {

const tid = this.tid > -1 ? this.tid : Process.getCurrentThreadId();

const self = this;

let pExtra = {};

console.log("[STARTING TRACE] UID=" + this.uid + "Thread" + tid);

this.filterModuleScope();

const opts = {

events: {

call: true

},

transform: function (iterator) {

let instruction;

let next = 0;

let threadExtra = pExtra;

threadExtra.hookAfter = null;

while ((instruction = iterator.next()) !== null) {

next = 1;

next = self.trace(iterator, instruction, threadExtra);

if (next == -1) {

continue;

}

if (next > 0) {

iterator.keep();

}

}

}

};

if (this.isTrackCoverage()) {

console.log("TRACK COVERAGE");

opts.events.compile = true;

opts.onReceive = (pEvents) => {

this.processBbsCoverage(Stalker.parse(pEvents, {

annotate: true,

stringify: false,

}));

};

this.coverage.initOutput();

}

Stalker.follow(tid, opts);

}

}

exports.InterruptorAgent = InterruptorAgent;

InterruptorAgent.FLAVOR_DXC = "dxc";

InterruptorAgent.FLAVOR_STRACE = "strace";

},{"../utilities/Coverage":10,"./InterruptorException":9}],9:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.InterruptorGenericException = exports.MonitoredError = exports.ErrorCode = void 0;

var ErrorCode;

(function (ErrorCode) {

ErrorCode[ErrorCode["GENERIC"] = 1000] = "GENERIC";

})(ErrorCode = exports.ErrorCode || (exports.ErrorCode = {}));

class MonitoredError extends Error {

constructor(pCmp, pMsg, pCode = null, pExtra = null) {

super(pMsg);

this.cmp = pCmp;

this.code = pCode;

this.extra = pExtra;

}

getCode() {

return this.code;

}

getExtra() {

return this.extra;

}

toString() {

return `[${this.cmp}] [#${this.code != null ? this.code : "<null>"} ${this.message}`;

}

toObject(pIncludeExtra = false) {

return {

cmp: this.cmp,

code: this.code,

msg: this.message,

extra: pIncludeExtra ? this.extra : null

};

}

}

exports.MonitoredError = MonitoredError;

class InterruptorGenericException extends MonitoredError {

constructor(pMsg, pCode = null, pExtra = null) {

super('GLOBAL', pMsg, pCode, pExtra);

}

}

exports.InterruptorGenericException = InterruptorGenericException;

InterruptorGenericException.ERR = {

INVALID_PID: ErrorCode.GENERIC + 101,

INVALID_TID: ErrorCode.GENERIC + 102,

UKNOW_SYSCALL: ErrorCode.GENERIC + 103,

};

InterruptorGenericException.INVALID_PID = () => { return new InterruptorGenericException("PID is invalid", InterruptorGenericException.ERR.INVALID_PID); };

InterruptorGenericException.INVALID_TID = () => { return new InterruptorGenericException("Thread ID is invalid", InterruptorGenericException.ERR.INVALID_TID); };

InterruptorGenericException.UNKNOW_SYSCALL = (sys) => { return new InterruptorGenericException("Syscall'" + sys + "'not exists", InterruptorGenericException.ERR.UKNOW_SYSCALL); };

},{}],10:[function(require,module,exports){

"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

exports.CoverageAgent = void 0;

class CoverageAgent {

constructor(pInterruptor) {

this.enabled = true;

this.interruptor = null;

this.flavor = "dr";

this.fname = null;

this.events = new Map();

this.threads = [];

this.onCoverage = () => { };

this.out = null;

this.stops = { count: Infinity };

this.interruptor = pInterruptor;

}

static from(pConfig, pInterruptor) {

const agent = new CoverageAgent(pInterruptor);

for (let i in pConfig) {

switch (i) {

case 'fname':

agent.fname = pConfig.fname;

break;

case 'enabled':

agent.enabled = pConfig.enabled;

break;

case 'format':

agent.flavor = pConfig.flavor;

break;

case 'stops':

agent.stops = pConfig.stops;

break;

case 'onCoverage':

agent.onCoverage = pConfig.onCoverage;

break;

}

}

return agent;

}

initOutput() {

if (this.fname != null) {

this.out = new File(this.fname, "wb+");

console.log("[COVERAGE] Create file :" + this.fname);

}

}

emit(coverageData) {

(this.onCoverage)(coverageData);

if (this.out != null) {

try {

this.out.write(coverageData);

}

catch (e) {

}

}

}

processStalkerEvent(pEvent) {

const type = pEvent[CoverageAgent.COMPILE_EVENT_TYPE_INDEX];

if (type.toString() === CoverageAgent.COMPILE_EVENT_TYPE.toString()) {

const start = pEvent[CoverageAgent.COMPILE_EVENT_START_INDEX];

const end = pEvent[CoverageAgent.COMPILE_EVENT_END_INDEX];

this.events.set(start, end);

if (this.isStopReached()) {

this.stop();

}

}

}

isStopReached() {

return (this.events.size >= this.stops.count);

}

isStepReached() {

return (this.stops.step > -1) && (this.events.size % this.stops.step == 0);

}

static convertString(data) {

const buf = new ArrayBuffer(data.length);

const view = new Uint8Array(buf);

for (let i = 0; i < data.length; i += 1) {

view[i] = data.charCodeAt(i);

}

return buf;

}

static padStart(data, length, pad) {

const paddingLength = length - data.length;

const partialPadLength = paddingLength % pad.length;

const fullPads = paddingLength - partialPadLength / pad.length;

const result = pad.repeat(fullPads) + pad.substring(0, partialPadLength)

+ data;

return result;

}

static write16le(address, value) {

let i;

for (i = 0; i < CoverageAgent.BYTES_PER_U16; i += 1) {

const byteValue = (value >> (CoverageAgent.BITS_PER_BYTE * i)) & CoverageAgent.BYTE_MASK;

address.add(i)

.writeU8(byteValue);

}

}

static write32le(address, value) {

let i;

for (i = 0; i < CoverageAgent.BYTES_PER_U32; i += 1) {

const byteValue = (value >> (CoverageAgent.BITS_PER_BYTE * i)) & CoverageAgent.BYTE_MASK;

address.add(i)

.writeU8(byteValue);

}

}

stop() {

const eventList = Array.from(this.events.entries());

const convertedEvents = eventList.map(([start, end]) => this.convertEvent(start, end));

const nonNullEvents = convertedEvents.filter((e) => e !== undefined);

this.emitHeader(nonNullEvents.length);

for (const convertedEvent of nonNullEvents) {

if (convertedEvent !== undefined) {

this.emitEvent(convertedEvent);

}

}

if (this.out != null) {

this.out.close();

this.out = null;

console.warn("[COVERAGE] Output file" + this.fname + "closed !");

}

}

convertEvent(start, end) {

for (let i = 0; i < this.interruptor.modules.length; i += 1) {

const base = this.interruptor.modules[i].base;

const size = this.interruptor.modules[i].size;

const limit = base.add(size);

if (start.compare(base) < 0) {

continue;

}

if (end.compare(limit) > 0) {

continue;

}

const offset = start.sub(base)

.toInt32();

const length = end.sub(start)

.toInt32();

if (!this.isInRange(base, start, end)) {

return undefined;

}

const event = {

length,

moduleId: i,

offset,

};

return event;

}

return undefined;

}

emitEvent(event) {

const memory = Memory.alloc(CoverageAgent.EVENT_TOTAL_SIZE);

CoverageAgent.write32le(memory.add(CoverageAgent.EVENT_START_OFFSET), event.offset);

CoverageAgent.write16le(memory.add(CoverageAgent.EVENT_SIZE_OFFSET), event.length);

CoverageAgent.write16le(memory.add(CoverageAgent.EVENT_MODULE_OFFSET), event.moduleId);

const buf = ArrayBuffer.wrap(memory, CoverageAgent.EVENT_TOTAL_SIZE);

this.emit(buf);

}

emitHeader(events) {

this.emit(CoverageAgent.convertString("DRCOV VERSION: 2\n"));

this.emit(CoverageAgent.convertString("DRCOV FLAVOR: frida\n"));

this.emit(CoverageAgent.convertString(`Module Table: version 2, count ${this.interruptor.modules.length}\n`));

this.emit(CoverageAgent.convertString("Columns: id, base, end, entry, checksum, timestamp, path\n"));

this.interruptor.modules.forEach((m, idx) => {

this.emitModule(idx, m);

});

this.emit(CoverageAgent.convertString(`BB Table: ${events} bbs\n`));

}

emitModule(idx, module) {

const moduleId = CoverageAgent.padStart(idx.toString(), CoverageAgent.COLUMN_WIDTH_MODULE_ID, " ");

let base = module.base

.toString(16);

base = CoverageAgent.padStart(base, CoverageAgent.COLUMN_WIDTH_MODULE_BASE, "0");

let end = module.base

.add(module.size)

.toString(16);

end = CoverageAgent.padStart(end, CoverageAgent.COLUMN_WIDTH_MODULE_END, "0");

const entry = "0".repeat(CoverageAgent.COLUMN_WIDTH_MODULE_ENTRY);

const checksum = "0".repeat(CoverageAgent.COLUMN_WIDTH_MODULE_CHECKSUM);

const timeStamp = "0".repeat(CoverageAgent.COLUMN_WIDTH_MODULE_TIMESTAMP);

const path = module.path;

const elements = [moduleId, base, end, entry, checksum, timeStamp, path];

const line = elements.join(",");

this.emit(CoverageAgent.convertString(`${line}\n`));

}

isInRange(base, start, end) {

const ranges = this.interruptor.ranges.get(base);

if (ranges === undefined) {

return false;

}

for (const range of ranges) {

if (end.compare(range.base) < 0) {

continue;

}

const limit = range.base.add(range.size);

if (start.compare(limit) >= 0) {

continue;

}

return true;

}

return false;

}

}

exports.CoverageAgent = CoverageAgent;

CoverageAgent.BITS_PER_BYTE = 8;

CoverageAgent.BYTE_MASK = 0xFF;

CoverageAgent.BYTES_PER_U16 = 2;

CoverageAgent.BYTES_PER_U32 = 4;

CoverageAgent.COLUMN_WIDTH_MODULE_BASE = 16;

CoverageAgent.COLUMN_WIDTH_MODULE_CHECKSUM = 16;

CoverageAgent.COLUMN_WIDTH_MODULE_END = 16;

CoverageAgent.COLUMN_WIDTH_MODULE_ENTRY = 16;

CoverageAgent.COLUMN_WIDTH_MODULE_ID = 3;

CoverageAgent.COLUMN_WIDTH_MODULE_TIMESTAMP = 8;

CoverageAgent.COMPILE_EVENT_END_INDEX = 2;

CoverageAgent.COMPILE_EVENT_START_INDEX = 1;

CoverageAgent.COMPILE_EVENT_TYPE = "compile";

CoverageAgent.COMPILE_EVENT_TYPE_INDEX = 0;

CoverageAgent.EVENT_MODULE_OFFSET = 6;

CoverageAgent.EVENT_SIZE_OFFSET = 4;

CoverageAgent.EVENT_START_OFFSET = 0;

CoverageAgent.EVENT_TOTAL_SIZE = 8;

},{}],11:[function(require,module,exports){

var Interruptor = require('../dist/index.js').default.LinuxArm64();

Java.perform(()=>{

Interruptor

.newAgentTracer({

exclude: {

syscalls: ["clock_gettime"]

}

})

.start();

})

},{"../dist/index.js":1}]},{},[11])