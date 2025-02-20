## 简介

与内核 Livepatch 相关的一些核心结构、宏和函数。Livepatch 允许在内核运行时动态修改函数和数据.

## core.h

### 宏定义

```c
// safe 版本用于在遍历过程中修改链表（即删除当前元素时不会导致遍历出错）
#define klp_for_each_patch_safe(patch, tmp_patch)		\
	list_for_each_entry_safe(patch, tmp_patch, &klp_patches, list)

// 遍历 klp_patches 链表中的每个补丁
#define klp_for_each_patch(patch)	\
	list_for_each_entry(patch, &klp_patches, list)
```

### 函数声明

```c
void klp_free_patch_async(struct klp_patch *patch); // 异步释放补丁
void klp_free_replaced_patches_async(struct klp_patch *new_patch); // 释放已经被替换的补丁，异步进行
void klp_unpatch_replaced_patches(struct klp_patch *new_patch); // 撤销替换的补丁
void klp_discard_nops(struct klp_patch *new_patch); // 丢弃由补丁产生的 NOP（No Operation）指令
```

### 内联函数

```c
// 检查 klp_object 是否已加载
static inline bool klp_is_object_loaded(struct klp_object *obj)
{
    // 如果是vmlinux不是模块则返回false
	return !obj->name || obj->mod;
}


// 在应用补丁之前调用回调函数（pre_patch）. 如果回调函数存在，它会被执行.
static inline int klp_pre_patch_callback(struct klp_object *obj)
{
	int ret = 0;

	if (obj->callbacks.pre_patch)
		ret = (*obj->callbacks.pre_patch)(obj);

    // post_unpatch_enabled 字段根据回调返回值来设置，用于控制是否启用卸载补丁后的回调
	obj->callbacks.post_unpatch_enabled = !ret;

	return ret;
}

// 在应用补丁后调用回调函数（post_patch）
static inline void klp_post_patch_callback(struct klp_object *obj)
{
	if (obj->callbacks.post_patch)
		(*obj->callbacks.post_patch)(obj);
}

// 在卸载补丁后调用回调函数（post_unpatch）
static inline void klp_post_unpatch_callback(struct klp_object *obj)
{
	if (obj->callbacks.post_unpatch_enabled &&
	    obj->callbacks.post_unpatch)
		(*obj->callbacks.post_unpatch)(obj);

    // 这里无论成不成功启用卸载后的回调都置为 false
	obj->callbacks.post_unpatch_enabled = false;
}
```

## core.c

### 流程解析

1. 补丁启用流程

    用户调用 klp_enable_patch():

    - 检查模块是否为 Livepatch 模块（is_livepatch_module()）.

    - 初始化补丁数据结构（klp_init_patch_early()）.

    - 创建 Sysfs 接口（kobject_add()）.

    - 应用重定位（klp_apply_object_relocs()）.

    - 注册 ftrace 钩子（klp_patch_object()）.

    - 启动状态过渡（klp_start_transition()）.

2. 状态过渡

    - 通过 klp_transition_patch 跟踪当前补丁.

    - 使用 RCU 和任务检查确保安全切换.

    - 最终通过 klp_try_complete_transition() 完成过渡.

    ```c
    /**
    * klp_enable_patch() - enable the livepatch
    * @patch:	patch to be enabled
    *
    * Initializes the data structure associated with the patch, creates the sysfs
    * interface, performs the needed symbol lookups and code relocations,
    * registers the patched functions with ftrace.
    *
    * This function is supposed to be called from the livepatch module_init()
    * callback.
    *
    * Return: 0 on success, otherwise error
    */
    int klp_enable_patch(struct klp_patch *patch)
    {
        int ret;
        struct klp_object *obj;

        if (!patch || !patch->mod || !patch->objs)
            return -EINVAL;

        klp_for_each_object_static(patch, obj) {
            if (!obj->funcs)
                return -EINVAL;
        }

        // CONFIG_LIVEPATCH 宏开启.
        if (!is_livepatch_module(patch->mod)) {
            pr_err("module %s is not marked as a livepatch module\n",
                patch->mod->name);
            return -EINVAL;
        }

        if (!klp_initialized())
            return -ENODEV;

        // CONFIG_HAVE_RELIABLE_STACKTRACE 或者 CONFIG_STACKTRACE 使能
        if (!klp_have_reliable_stack()) {
            pr_warn("This architecture doesn't have support for the livepatch consistency model.\n");
            pr_warn("The livepatch transition may never complete.\n");
        }

        mutex_lock(&klp_mutex);

        if (!klp_is_patch_compatible(patch)) {
            pr_err("Livepatch patch (%s) is not compatible with the already installed livepatches.\n",
                patch->mod->name);
            mutex_unlock(&klp_mutex);
            return -EINVAL;
        }

        if (!try_module_get(patch->mod)) {
            mutex_unlock(&klp_mutex);
            return -ENODEV;
        }

        klp_init_patch_early(patch);

        ret = klp_init_patch(patch);
        if (ret)
            goto err;

        ret = __klp_enable_patch(patch);
        if (ret)
            goto err;

        mutex_unlock(&klp_mutex);

        return 0;

    err:
        klp_free_patch_start(patch);

        mutex_unlock(&klp_mutex);

        klp_free_patch_finish(patch);

        return ret;
    }
    ```

## 其他函数

这里有时间慢慢地把全部文件的函数都列出来看一遍.

```c
// 检查 klp_object 是否为一个内核模块
static bool klp_is_module(struct klp_object *obj)
{
	return obj->name;
}

// 接受一个字符串参数 name，它是内核模块的名称，然后填充 obj->mod
static void klp_find_object_module(struct klp_object *obj)
{
	struct module *mod;

	if (!klp_is_module(obj))
		return;

	rcu_read_lock_sched();

    // 必须在 RCU（Read-Copy-Update）调度临界区内调用
	mod = find_module(obj->name);

	if (mod && mod->klp_alive)
		obj->mod = mod;

	rcu_read_unlock_sched();
}

// 检查 Livepatch 是否已初始化. 如果 klp_root_kobj 非空，说明 Livepatch 已初始化，否则返回 false.
static bool klp_initialized(void)
{
	return !!klp_root_kobj;
}

// klp_object 中查找与 old_func 匹配的功能（klp_func）
static struct klp_func *klp_find_func(struct klp_object *obj,
				      struct klp_func *old_func)
{
	struct klp_func *func;

	klp_for_each_func(obj, func) {
		if ((strcmp(old_func->old_name, func->old_name) == 0) &&
		    (old_func->old_sympos == func->old_sympos)) {
			return func;
		}
	}

	return NULL;
}

// klp_patch 中查找与 old_obj 匹配的功能（klp_obj）
static struct klp_object *klp_find_object(struct klp_patch *patch,
					  struct klp_object *old_obj)
{
	struct klp_object *obj;

	klp_for_each_object(patch, obj) {
		if (klp_is_module(old_obj)) {
			if (klp_is_module(obj) &&
			    strcmp(old_obj->name, obj->name) == 0) {
				return obj;
			}
		} else if (!klp_is_module(obj)) {
			return obj;
		}
	}

	return NULL;
}

// 在内核符号表中查找特定符号的位置，支持指定符号的多个定义位置（通过 sympos 参数）。通过遍历符号表，它能够处理符号的匹配、符号位置的验证以及可能的多重符号定义

// 1. 这个回调函数用于匹配地址. 它检查当前地址是否满足特定的条件，例如，如果已经找到了指定位置的符号，或者对于没有位置要求的符号，找到多个匹配项时就返回 1，表示匹配成功. 否则，返回 0，继续查找.
static int klp_match_callback(void *data, unsigned long addr)
{
    struct klp_find_arg *args = data;

    args->addr = addr;
    args->count++;

    if ((args->pos && (args->count == args->pos)) ||
        (!args->pos && (args->count > 1)))
        return 1;

    return 0;
}

// 2. 处理每个符号的查找过程
static int klp_find_callback(void *data, const char *name, unsigned long addr)
{
	struct klp_find_arg *args = data;

	if (strcmp(args->name, name))
		return 0;

	return klp_match_callback(data, addr);
}

// 3. 实际执行符号查找的函数。它通过遍历符号表查找匹配的符号，并返回符号的地址.
static int klp_find_object_symbol(const char *objname, const char *name,
				  unsigned long sympos, unsigned long *addr)
{
	struct klp_find_arg args = {
		.name = name,
		.addr = 0,
		.count = 0,
		.pos = sympos,
	};

	if (objname)
		module_kallsyms_on_each_symbol(objname, klp_find_callback, &args);
	else
		kallsyms_on_each_match_symbol(klp_match_callback, name, &args);

	/*
	 * Ensure an address was found. If sympos is 0, ensure symbol is unique;
	 * otherwise ensure the symbol position count matches sympos.
	 */
    // 在这里 count 需要和 sympos 匹配，内核中可能存在多个同名的符号
	if (args.addr == 0)
		pr_err("symbol '%s' not found in symbol table\n", name);
	else if (args.count > 1 && sympos == 0) {
		pr_err("unresolvable ambiguity for symbol '%s' in object '%s'\n",
		       name, objname);
	} else if (sympos != args.count && sympos > 0) {
		pr_err("symbol position %lu for symbol '%s' in object '%s' not found\n",
		       sympos, name, objname ? objname : "vmlinux");
	} else {
		*addr = args.addr;
		return 0;
	}

	*addr = 0;
	return -EINVAL;
}


```