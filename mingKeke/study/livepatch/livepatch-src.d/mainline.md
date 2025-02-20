## 简介

本人是小白，该文档希望能够在学习livepatch的过程中记录一下对的源码分析过程，可能分析有误，欢迎共同探讨.

邮箱地址: 604631736@qq.com

## 流程分析

```bash
加载补丁模块
   │
   ▼
klp_init_patch_early (初始化基础结构)
   │
   ▼
klp_init_patch (注册到sysfs，加入全局链表)
   │
   ▼
klp_enable_patch (启用补丁)
   ├─► klp_patch_object (遍历对象)
   │    ├─► klp_patch_func (应用单函数补丁)
   │    │     ├─► 创建/查找 klp_ops
   │    │     ├─► 注册 ftrace 回调
   │    │     └─► 压入函数栈
   │    └─► 若失败，回滚整个对象
   │
   ▼
运行时函数调用
   │
   ▼
klp_ftrace_handler (拦截并跳转到补丁函数)
   │
   ▼
补丁代码生效



禁用补丁
   │
   ▼
klp_unpatch_objects (遍历对象撤销补丁)
   ├─► klp_unpatch_object (撤销对象补丁)
   │    └─► klp_unpatch_func (弹出函数栈)
   │         ├─► 若栈空，注销 ftrace
   │         └─► 释放 klp_ops
   └─► 删除 sysfs 节点，释放资源
```

## 源码分析

### Makefile

livepatch 的Makefile在 `kernel/livepatch/Makefile`

```makefile
# SPDX-License-Identifier: GPL-2.0-only
obj-$(CONFIG_LIVEPATCH) += livepatch.o # CONFIG_LIVEPATCH 被启用则 livepatch.o 会被编译进内核中

livepatch-objs := core.o patch.o shadow.o state.o transition.o # 这些是构建 livepatch.o 的依赖文件，下面需要挨个儿去分析一下.
```

### Datastructures

数据结构定义在 `include/linux/livepatch.h`

我写的 [livepatch](../livepatch.md) 也有简单介绍数据结构之间的关系.

klp_object

```c
/**
 * struct klp_object - kernel object structure for live patching
 * @name:	module name (or NULL for vmlinux)
 * @funcs:	function entries for functions to be patched in the object
 * @callbacks:	functions to be executed pre/post (un)patching
 * @kobj:	kobject for sysfs resources
 * @func_list:	dynamic list of the function entries
 * @node:	list node for klp_patch obj_list
 * @mod:	kernel module associated with the patched object
 *		(NULL for vmlinux)
 * @dynamic:    temporary object for nop functions; dynamically allocated
 * @patched:	the object's funcs have been added to the klp_ops list
 */
struct klp_object {
	/* external */
	const char *name;
	struct klp_func *funcs;
	struct klp_callbacks callbacks;

	/* internal */
	struct kobject kobj;
	struct list_head func_list;
	struct list_head node;
	struct module *mod;
	bool dynamic;
	bool patched;
};
```

klp_func

```c
/**
 * struct klp_func - function structure for live patching
 * @old_name:	name of the function to be patched
 * @new_func:	pointer to the patched function code
 * @old_sympos: a hint indicating which symbol position the old function
 *		can be found (optional)
 * @old_func:	pointer to the function being patched
 * @kobj:	kobject for sysfs resources
 * @node:	list node for klp_object func_list
 * @stack_node:	list node for klp_ops func_stack list
 * @old_size:	size of the old function
 * @new_size:	size of the new function
 * @nop:        temporary patch to use the original code again; dyn. allocated
 * @patched:	the func has been added to the klp_ops list
 * @transition:	the func is currently being applied or reverted
 *
 * The patched and transition variables define the func's patching state.  When
 * patching, a func is always in one of the following states:
 *
 *   patched=0 transition=0: unpatched
 *   patched=0 transition=1: unpatched, temporary starting state
 *   patched=1 transition=1: patched, may be visible to some tasks
 *   patched=1 transition=0: patched, visible to all tasks
 *
 * And when unpatching, it goes in the reverse order:
 *
 *   patched=1 transition=0: patched, visible to all tasks
 *   patched=1 transition=1: patched, may be visible to some tasks
 *   patched=0 transition=1: unpatched, temporary ending state
 *   patched=0 transition=0: unpatched
 */
struct klp_func {
	/* external */
	const char *old_name;
	void *new_func;
	/*
	 * The old_sympos field is optional and can be used to resolve
	 * duplicate symbol names in livepatch objects. If this field is zero,
	 * it is expected the symbol is unique, otherwise patching fails. If
	 * this value is greater than zero then that occurrence of the symbol
	 * in kallsyms for the given object is used.
	 */
	unsigned long old_sympos;

	/* internal */
	void *old_func;
	struct kobject kobj;
	struct list_head node; // 用于将 klp_func 结构体插入到 klp_object 的 func_list 中
	struct list_head stack_node; // 用于将 klp_func 插入到 klp_ops 的 func_stack 列表中
	unsigned long old_size, new_size;
	bool nop;

    /*
    控制补丁的状态
    patched = 0, transition = 0：表示该函数没有被补丁.
    patched = 0, transition = 1：表示该函数已经开始补丁，但补丁未完全应用，处于临时状态.
    patched = 1, transition = 1：表示该函数已经被补丁，但补丁可能还未完全生效，或者部分任务可能仍能看到旧函数.
    patched = 1, transition = 0：表示该函数已经完全被补丁，并且所有任务都能看到新函数.
    当撤销补丁时，状态将反向过渡：
    patched = 1, transition = 0：补丁已完全应用，并且所有任务都能看到新函数.
    patched = 1, transition = 1：补丁仍在撤销过程中.
    patched = 0, transition = 1：补丁正在撤销，临时恢复状态.
    patched = 0, transition = 0：函数完全恢复为原始状态.
    };
    */
	bool patched; // 控制 patch 的应用状态
	bool transition; // 表示 patch 的过渡状态
```

klp_patch

```c
/**
 * struct klp_patch - patch structure for live patching
 * @mod:	reference to the live patch module
 * @objs:	object entries for kernel objects to be patched
 * @states:	system states that can get modified
 * @replace:	replace all actively used patches
 * @list:	list node for global list of actively used patches
 * @kobj:	kobject for sysfs resources
 * @obj_list:	dynamic list of the object entries
 * @enabled:	the patch is enabled (but operation may be incomplete)
 * @forced:	was involved in a forced transition
 * @free_work:	patch cleanup from workqueue-context
 * @finish:	for waiting till it is safe to remove the patch module
 */
struct klp_patch {
	/* external */
	struct module *mod;
	struct klp_object *objs;
	struct klp_state *states;
	bool replace;

	/* internal */
	struct list_head list;
	struct kobject kobj; // 通过这个 kobject，可以将与该补丁相关的信息暴露到 sysfs，从而允许用户空间访问和控制补丁.
	struct list_head obj_list;
	bool enabled;
	bool forced;
	struct work_struct free_work;
	struct completion finish;
};
```

klp_state

```c
/**
 * struct klp_state - state of the system modified by the livepatch
 * @id:		system state identifier (non-zero)
 * @version:	version of the change
 * @data:	custom data
 */
struct klp_state {
	unsigned long id;
	unsigned int version;
	void *data;
};
```

### core

[core](core.md)

### patch

[patch](patch.md)