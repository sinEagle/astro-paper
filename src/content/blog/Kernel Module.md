---
title: Kernel Module
author: sineagle
pubDatetime: 2024-11-20T16:26:32
modDatetime: 
featured: false
draft: false
tags:
  - Linux
  - debug
description: 内核模块机制
---
## Table of contents

## 内核模块构成

Linux模块机制支持运行时动态加载或卸载一个模块，不需要重新编译重启内核，这种是可加载模块，另一种是内置模块是编译到内核里的。

• module_init(hello_init)： 模块加载函数，指定入口函数（入口函数完成初始化工作）

• module_exit(hello_exit)： 模块卸载函数，指定入口函数（卸载时自动执行）

• MODULE_LICENSE("GPL")：模块许可声明

• module_param：模块参数

• MODULE_PARAM_DESC： 模块参数描述

• MODULE_AUTHOR：模块作者

• MODULE_DESCRIPTION： 模块描述信息

• EXPORT_SYMBOL：导出全局符号

...

## 内核污染

内核一般都有模块许可证，内核是GPL权限，编写模块不进行声明会有tained警告，此时就是受污染的

```bash
 cat /proc/sys/kernel/tainted # 查看是否受污染
```

## 模块的Makefile分析

obj-m: 不会编译到内核，但是会生成一个独立的ko文件，可加载模块
obj-y: 编译到内核中

```bash
.PHONY: all clean

#ifneq ($(KERNELRELEASE),)
obj-m := hello.o

#else

EXTRA_CFLAGS += -DDEBUG
KDIR := /home/lab/kernel/linux-5.10.99

all:
                        make -C $(KDIR) M=$(PWD) modules
clean:
                        make -C $(KDIR) M=$(PWD) modules clean

#endif

```

在当前目录执行make时，由于找不到KERNELRELEASE，会执行else下面，在执行到-C时，会到内核源码目录解析Makefile，这时候该变量有值了，然后回到当前目录二次解析，最后执行obj-m := hello.o

## 模块的运行过程

编写hello.c

```c
#include <linux/init.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");

static char* name = "World";
module_param(name, charp, 0660);
MODULE_PARM_DESC(name, "Whom this module say hello to");

static int __init hello_init(void)
{
        printk("Hello %s\n", name);
        dump_stack();
        return 0;
}

static void __exit hello_exit(void)
{
        printk("Good Bye %s!\n", name);
        dump_stack();
}

module_init(hello_init);
module_exit(hello_exit);
```

这里的module_init实际上是给hello_init起了一个别名，叫做init_module

```cpp
/* Each module must use one module_init(). */
#define module_init(initfn)                                     \
        static inline initcall_t __maybe_unused __inittest(void)                \
        { return initfn; }                                      \
        int init_module(void) __copy(initfn) __attribute__((alias(#initfn)));

/* This is only required if you want to be unloadable. */
#define module_exit(exitfn)                                     \
        static inline exitcall_t __maybe_unused __exittest(void)                \
        { return exitfn; }                                      \
        void cleanup_module(void) __copy(exitfn) __attribute__((alias(#exitfn)));


```

Makefile

```bash
.PHONY: all clean
        #ifneq ($(KERNELRELEASE),)

obj-m := hello.o

#else

EXTRA_CFLAGS += -DDEBUG
KDIR := /home/lab/tftpboot/kernel/linux-5.10.99
DEST := /home/lab/nfs/module/

all:
                        make -C $(KDIR) M=$(PWD) modules
clean:
                        make -C $(KDIR) M=$(PWD) modules clean

#endif

```

生成的有个文件叫做hello.mod.c，在gnu.linkonce.this_module段中存放`module`结构体变量，包括入口函数地址、结束函数地址等

```cpp
#include <linux/module.h>
#define INCLUDE_VERMAGIC
#include <linux/build-salt.h>
#include <linux/vermagic.h>
#include <linux/compiler.h>

BUILD_SALT;

MODULE_INFO(vermagic, VERMAGIC_STRING);
MODULE_INFO(name, KBUILD_MODNAME);

__visible struct module __this_module
__section(".gnu.linkonce.this_module") = {
        .name = KBUILD_MODNAME,
        .init = init_module,
#ifdef CONFIG_MODULE_UNLOAD
        .exit = cleanup_module,
#endif
        .arch = MODULE_ARCH_INIT,
};

#ifdef CONFIG_RETPOLINE
MODULE_INFO(retpoline, "Y");
#endif

MODULE_INFO(depends, "");

```

```bash
root@ubuntu ~/s/module# readelf -S hello.ko -W
There are 26 section headers, starting at offset 0xd18:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .note.gnu.build-id NOTE            0000000000000000 000040 000024 00   A  0   0  4
  [ 2] .note.Linux       NOTE            0000000000000000 000064 000018 00   A  0   0  4
  [ 3] .text             PROGBITS        0000000000000000 00007c 000000 00  AX  0   0  1
  [ 4] .init.text        PROGBITS        0000000000000000 00007c 00001b 00  AX  0   0  1
  [ 5] .rela.init.text   RELA            0000000000000000 000a60 000060 18   I 23   4  8
  [ 6] .exit.text        PROGBITS        0000000000000000 000097 000018 00  AX  0   0  1
  [ 7] .rela.exit.text   RELA            0000000000000000 000ac0 000060 18   I 23   6  8
  [ 8] .rodata.str1.1    PROGBITS        0000000000000000 0000af 00001e 01 AMS  0   0  1
  [ 9] .modinfo          PROGBITS        0000000000000000 0000cd 000089 00   A  0   0  1
  [10] __param           PROGBITS        0000000000000000 000158 000028 00   A  0   0  8
  [11] .rela__param      RELA            0000000000000000 000b20 000060 18   I 23  10  8
  [12] .rodata           PROGBITS        0000000000000000 000180 000005 00   A  0   0  1
  [13] .orc_unwind_ip    PROGBITS        0000000000000000 000185 000010 00   A  0   0  1
  [14] .rela.orc_unwind_ip RELA            0000000000000000 000b80 000060 18   I 23  13  8
  [15] .orc_unwind       PROGBITS        0000000000000000 000195 000018 00   A  0   0  1
  [16] .data             PROGBITS        0000000000000000 0001b0 000008 00  WA  0   0  8
  [17] .rela.data        RELA            0000000000000000 000be0 000018 18   I 23  16  8
  [18] .gnu.linkonce.this_module PROGBITS        0000000000000000 0001c0 000380 00  WA  0   0 64
  [19] .rela.gnu.linkonce.this_module RELA            0000000000000000 000bf8 000030 18   I 23  18  8
  [20] .bss              NOBITS          0000000000000000 000540 000000 00  WA  0   0  1
  [21] .comment          PROGBITS        0000000000000000 000540 000058 01  MS  0   0  1
  [22] .note.GNU-stack   PROGBITS        0000000000000000 000598 000000 00      0   0  1
  [23] .symtab           SYMTAB          0000000000000000 000598 000390 18     24  32  8
  [24] .strtab           STRTAB          0000000000000000 000928 000135 00      0   0  1
  [25] .shstrtab         STRTAB          0000000000000000 000c28 0000ef 00      0   0  1

```

观察对应的函数调用栈

```bash
[root@sineagle_x64 module]# insmod hello.ko
[19998.164614] Hello World
[19998.168117] CPU: 0 PID: 151 Comm: insmod Tainted: P           O      5.10.99 #1
[19998.168377] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.13.0-48-gd9c812dda519-prebuilt.qemu.org 04/01/2014
[19998.169118] Call Trace:
[19998.172875]  dump_stack+0x57/0x6a
[19998.174079]  ? 0xffffffffc00ce000
[19998.174894]  hello_init+0x18/0x1000 [hello]
[19998.175349]  do_one_initcall+0x3f/0x1b0
[19998.176744]  ? free_vmap_area_noflush+0x8d/0xe0
[19998.177234]  ? _cond_resched+0x10/0x20
[19998.177437]  ? kmem_cache_alloc_trace+0x3a/0x1b0
[19998.178033]  do_init_module+0x56/0x200
[19998.178550]  load_module+0x234f/0x2690
[19998.179020]  ? __alloc_pages_nodemask+0x162/0x2c0
[19998.179642]  ? __do_sys_init_module+0x159/0x190
[19998.180144]  __do_sys_init_module+0x159/0x190
[19998.181318]  do_syscall_64+0x33/0x40
[19998.182032]  entry_SYSCALL_64_after_hwframe+0x44/0xa9
[19998.184180] RIP: 0033:0x4c2b3d
[19998.184885] Code: 00 c3 66 2e 0f 1f 84 00 00 00 00 00 90 f3 0f 1e fa 48 89 f8 48 89 f7 48 89 d6 48 89 ca 4d 89 c2 4d 89 c8 4c 8b 4c 24 08 0f 05 <48> 3d 8
[19998.185858] RSP: 002b:00007ffd7ed220e8 EFLAGS: 00000246 ORIG_RAX: 00000000000000af
[19998.186568] RAX: ffffffffffffffda RBX: 00007ffd7ed22490 RCX: 00000000004c2b3d
[19998.187007] RDX: 00000000006701cb RSI: 0000000000001398 RDI: 0000000001fcfd30
[19998.187444] RBP: 0000000000000001 R08: 000000000000007c R09: 0000000000000049
[19998.187891] R10: 0000000000000001 R11: 0000000000000246 R12: 00000000006701cb
[19998.188361] R13: 00007ffd7ed22498 R14: 00000000006701cb R15: 0000000000000000

```

执行流程：do_syscall_64 -> __do_sys_init_module -> load_module -> do_init_module -> do_one_initcall -> hello_init

load_module 就是将我们的模块拷贝到内核空间里面, 进行签名认证、版本认证、申请内存等操作

```cpp
/* Allocate and load the module: note that size of section 0 is always
   zero, and we rely on this for optional sections. */
static int load_module(struct load_info *info, const char __user *uargs,
		       int flags)
{
	struct module *mod;
	long err = 0;
	char *after_dashes;

	/*
	 * Do the signature check (if any) first. All that
	 * the signature check needs is info->len, it does
	 * not need any of the section info. That can be
	 * set up later. This will minimize the chances
	 * of a corrupt module causing problems before
	 * we even get to the signature check.
	 *
	 * The check will also adjust info->len by stripping
	 * off the sig length at the end of the module, making
	 * checks against info->len more correct.
	 */
	err = module_sig_check(info, flags);
	if (err)
		goto free_copy;

	/*
	 * Do basic sanity checks against the ELF header and
	 * sections.
	 */
	err = elf_validity_check(info);
	if (err) {
		pr_err("Module has invalid ELF structures\n");
		goto free_copy;
	}

	/*
	 * Everything checks out, so set up the section info
	 * in the info structure.
	 */
	err = setup_load_info(info, flags);
	if (err)
		goto free_copy;

	/*
	 * Now that we know we have the correct module name, check
	 * if it's blacklisted.
	 */
	if (blacklisted(info->name)) {
		err = -EPERM;
		pr_err("Module %s is blacklisted\n", info->name);
		goto free_copy;
	}

	err = rewrite_section_headers(info, flags);
	if (err)
		goto free_copy;

	/* Check module struct version now, before we try to use module. */
	if (!check_modstruct_version(info, info->mod)) {
		err = -ENOEXEC;
		goto free_copy;
	}

	/* Figure out module layout, and allocate all the memory. */
	mod = layout_and_allocate(info, flags);
	if (IS_ERR(mod)) {
		err = PTR_ERR(mod);
		goto free_copy;
	}

	audit_log_kern_module(mod->name);
...........................

	/* Module is ready to execute: parsing args may do that. */
	after_dashes = parse_args(mod->name, mod->args, mod->kp, mod->num_kp,
				  -32768, 32767, mod,
				  unknown_module_param_cb);
	if (IS_ERR(after_dashes)) {
		err = PTR_ERR(after_dashes);
		goto coming_cleanup;
	} else if (after_dashes) {
		pr_warn("%s: parameters '%s' after `--' ignored\n",
		       mod->name, after_dashes);
	}

	/* Link in to sysfs. */
	err = mod_sysfs_setup(mod, info, mod->kp, mod->num_kp);
	if (err < 0)
		goto coming_cleanup;

	if (is_livepatch_module(mod)) {
		err = copy_module_elf(mod, info);
		if (err < 0)
			goto sysfs_cleanup;
	}

	/* Get rid of temporary copy. */
	free_copy(info);

	/* Done! */
	trace_module_load(mod);

	return do_init_module(mod);

```

之后执行do_init_module函数,主要进行初始化操作，获取hello_init函数，调用完后，进行释放相应的内存

freeinit->module_init = mod->init_layout.base

- 获取将init函数指针赋值，因为之前模块入口函数添加了`__init`, 会将该函数放到`.init.text`段中

do_one_initcall(mod->init)

ftrace_free_mem(mod, mod->init_layout.base, mod->init_layout.base +
mod->init_layout.size)

- 进行释放init函数的内存，因为这段初始化代码只运行一次，为了节省空间，进行释放

```cpp
static noinline int do_init_module(struct module *mod)
{
	int ret = 0;
	struct mod_initfree *freeinit;

	freeinit = kmalloc(sizeof(*freeinit), GFP_KERNEL);
	if (!freeinit) {
		ret = -ENOMEM;
		goto fail;
	}
	freeinit->module_init = mod->init_layout.base;

	/*
	 * We want to find out whether @mod uses async during init.  Clear
	 * PF_USED_ASYNC.  async_schedule*() will set it.
	 */
	current->flags &= ~PF_USED_ASYNC;

	do_mod_ctors(mod);
	/* Start the module */
	if (mod->init != NULL)
		ret = do_one_initcall(mod->init);
	if (ret < 0) {
		goto fail_free_freeinit;
	}
	if (ret > 0) {
		pr_warn("%s: '%s'->init suspiciously returned %d, it should "
			"follow 0/-E convention\n"
			"%s: loading module anyway...\n",
			__func__, mod->name, ret, __func__);
		dump_stack();
	}

	/* Now it's a first class citizen! */
	mod->state = MODULE_STATE_LIVE;
	blocking_notifier_call_chain(&module_notify_list,
				     MODULE_STATE_LIVE, mod);

	/* Delay uevent until module has finished its init routine */
	kobject_uevent(&mod->mkobj.kobj, KOBJ_ADD);

	/*
	 * We need to finish all async code before the module init sequence
	 * is done.  This has potential to deadlock.  For example, a newly
	 * detected block device can trigger request_module() of the
	 * default iosched from async probing task.  Once userland helper
	 * reaches here, async_synchronize_full() will wait on the async
	 * task waiting on request_module() and deadlock.
	 *
	 * This deadlock is avoided by perfomring async_synchronize_full()
	 * iff module init queued any async jobs.  This isn't a full
	 * solution as it will deadlock the same if module loading from
	 * async jobs nests more than once; however, due to the various
	 * constraints, this hack seems to be the best option for now.
	 * Please refer to the following thread for details.
	 *
	 * http://thread.gmane.org/gmane.linux.kernel/1420814
	 */
	if (!mod->async_probe_requested && (current->flags & PF_USED_ASYNC))
		async_synchronize_full();

	ftrace_free_mem(mod, mod->init_layout.base, mod->init_layout.base +
			mod->init_layout.size);
	mutex_lock(&module_mutex);
	/* Drop initial reference. */
	module_put(mod);
	trim_init_extable(mod);
#ifdef CONFIG_KALLSYMS
	/* Switch to core kallsyms now init is done: kallsyms may be walking! */
	rcu_assign_pointer(mod->kallsyms, &mod->core_kallsyms);
#endif
	module_enable_ro(mod, true);
	mod_tree_remove_init(mod);
	module_arch_freeing_init(mod);
	mod->init_layout.base = NULL;
	mod->init_layout.size = 0;
	mod->init_layout.ro_size = 0;
	mod->init_layout.ro_after_init_size = 0;
	mod->init_layout.text_size = 0;
	/*
	 * We want to free module_init, but be aware that kallsyms may be
	 * walking this with preempt disabled.  In all the failure paths, we
	 * call synchronize_rcu(), but we don't want to slow down the success
	 * path. module_memfree() cannot be called in an interrupt, so do the
	 * work and call synchronize_rcu() in a work queue.
	 *
	 * Note that module_alloc() on most architectures creates W+X page
	 * mappings which won't be cleaned up until do_free_init() runs.  Any
	 * code such as mark_rodata_ro() which depends on those mappings to
	 * be cleaned up needs to sync with the queued work - ie
	 * rcu_barrier()
	 */
	if (llist_add(&freeinit->node, &init_free_list))
		schedule_work(&init_free_wq);

	mutex_unlock(&module_mutex);
	wake_up_all(&module_wq);

	return 0;

```

然后调用do_one_initcall函数，根据传过来的fn（就是hello_init），就可以执行了

```cpp

int __init_or_module do_one_initcall(initcall_t fn)
{
	int count = preempt_count();
	char msgbuf[64];
	int ret;

	if (initcall_blacklisted(fn))
		return -EPERM;

	do_trace_initcall_start(fn);
	ret = fn();
	do_trace_initcall_finish(fn, ret);

	msgbuf[0] = 0;

	if (preempt_count() != count) {
		sprintf(msgbuf, "preemption imbalance ");
		preempt_count_set(count);
	}
	if (irqs_disabled()) {
		strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
		local_irq_enable();
	}
	WARN(msgbuf[0], "initcall %pS returned with %s\n", fn, msgbuf);

	add_latent_entropy();
	return ret;
}
```



```cpp
static int move_module(struct module *mod, struct load_info *info)
{
        int i;
        void *ptr;

        /* Do the allocs. */
        ptr = module_alloc(mod->core_layout.size);
        /*
         * The pointer to this block is stored in the module structure
         * which is inside the block. Just mark it as not being a
         * leak.
         */
        kmemleak_not_leak(ptr);
        if (!ptr)
                return -ENOMEM;

        memset(ptr, 0, mod->core_layout.size);
        mod->core_layout.base = ptr;
	    ....
}

```





__visible 宏：它是内核中用来显式标记某些符号的可见性，使得这些符号在内核模块中是可见的，特别是在内核模块之间的符号共享。

内核符号的导出意味着该符号可以被其他内核模块访问。内核符号导出通常使用 `EXPORT_SYMBOL` 或 `EXPORT_SYMBOL_GPL` 宏来实现。`__visible` 宏通常与 `EXPORT_SYMBOL` 一起使用，用于确保符号在内核模块加载时可见。

- `EXPORT_SYMBOL` 宏会将一个符号从内核模块中导出，允许其他模块访问它。如果你想将符号标记为可以导出的，可以将其与 `__visible` 宏一起使用。