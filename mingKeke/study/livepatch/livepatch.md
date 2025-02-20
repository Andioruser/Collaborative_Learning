## 简介

在很多情况下，用户不愿意重启系统. 这可能是因为他们的系统正在执行复杂的科学计算，或者在使用高峰期负载很重。 除了保持系统正常运行，用户还希望系统稳定安全。 实时补丁允许函数调用重定向，从而在不重启系统的情况下修复关键功能，为用户提供了这两方面的保障.

相应的使用示例参考: https://elixir.bootlin.com/linux/v6.14-rc2/source/samples/livepatch

### 一致性模型

Livepatch 的一致性模型是 kGraft 和 kpatch 的混合体：它使用了 kGraft 的每个任务一致性和系统调用障碍切换，并结合了 kpatch 的堆栈跟踪切换. 补丁以任务为单位，在任务被认为可以安全切换时打上. 启用补丁后，livepatch 会进入一个过渡状态，在这个状态下，任务会向补丁状态靠拢. 通常这种过渡状态可以在几秒钟内完成.

### 打补丁的时机

1. 第一种也是最有效的方法是对睡眠任务进行堆栈检查. 如果某个任务的堆栈中没有受影响的函数，就会对该任务打上补丁. 在大多数情况下，第一次尝试就会修补大部分或全部任务. 否则，它会定期尝试. 只有当体系结构具有可靠的堆栈（HAVE_RELIABLE_STACKTRACE）时，该选项才可用.

2. 第二种方法（如果需要）是内核退出切换. 当任务从系统调用、用户空间 IRQ 或信号返回用户空间时，任务会被切换. 它在以下情况下非常有用：修补在受影响函数上休眠的 I/O 绑定用户任务. 在这种情况下，您必须发送 SIGSTOP 和 SIGCONT，迫使它退出内核并打上补丁. 为 CPU 绑定的用户任务打上补丁. 如果任务高度绑定 CPU，那么它在下一次被 IRQ 中断时就会被打上补丁.

3. 对于空闲的 "swapper" 任务(https://blog.csdn.net/fervor_heart/article/details/8197851)，由于它们不会退出内核，因此在空闲循环中会调用 `klp_update_patch_state()`，这样就能在 CPU 进入空闲状态前对它们进行修补.

### sysfs操作

1. `/sys/kernel/livepatch/<patch>/transition` 文件会显示某个补丁是否处于过渡阶段。 同一时间只能有一个补丁处于过渡阶段。 如果有任务停留在初始补丁状态，补丁可以无限期地处于过渡状态。

2. `/proc/<pid>/patch_state` 文件，可用于确定哪些任务阻碍了修补操作的完成。 如果补丁处于过渡阶段，该文件显示 0 表示任务未打补丁，显示 1 表示任务已打补丁。 否则，如果没有补丁在过渡中，则显示-1。 可以用 SIGSTOP 和 SIGCONT 向任何阻碍过渡的任务发出信号，迫使它们改变补丁状态。 但这可能会对系统造成危害。

3. 如果在过渡过程中向 `/sys/kernel/livepatch/<patch>/enabled` 文件写入相反的值，就可以逆转并有效取消过渡。 然后，所有任务都将尝试返回到原始补丁状态。

4. 可以通过 `/sys/kernel/livepatch/<patch>/force` 属性影响转换. 在此写入 1 会清除所有任务的 `TIF_PATCH_PENDING` 标记，从而强制任务进入打补丁状态.

    > Important note! The force attribute is intended for cases when the transition gets stuck for a long time because of a blocking task. Administrator is expected to collect all necessary data (namely stack traces of such blocking tasks) and request a clearance from a patch distributor to force the transition. Unauthorized usage may cause harm to the system. It depends on the nature of the patch, which functions are (un)patched, and which functions the blocking tasks are sleeping in (/proc/<pid>/stack may help here). Removal (rmmod) of patch modules is permanently disabled when the force feature is used. It cannot be guaranteed there is no task sleeping in such module. It implies unbounded reference count if a patch module is disabled and enabled in a loop.

## Datastructures

1. [struct klp_func](https://www.kernel.org/doc/html/latest/livepatch/api.html#c.klp_func) 描述了特定函数的原始实现和新实现之间的关系.

2. [struct klp_object](https://www.kernel.org/doc/html/latest/livepatch/api.html#c.klp_object) 定义了同一对象中的补丁函数（struct klp_func）数组。 对象可以是 vmlinux (NULL) 或模块名，该结构有助于将每个对象的函数分组并一起处理.

3. [struct klp_patch](https://www.kernel.org/doc/html/latest/livepatch/api.html#c.klp_patch) 定义了一个补丁对象数组（struct klp_object）。 该结构能一致地处理所有补丁函数，并最终同步处理。 只有在找到所有打补丁的符号时，才会应用整个补丁.

## LivePatch的操作

实时补丁可以用五个基本操作来描述：加载、启用、替换、禁用、删除. 其中替换和禁用操作是相互排斥的. 它们对给定补丁的结果相同，但对系统的结果不同.

唯一合理的办法是在加载 livepatch 内核模块时启用补丁。 为此，必须在 `module_init()` 回调中调用 `klp_enable_patch()`。 主要原因有二：第一，只有模块才能方便地访问相关的 klp_patch 结构；第二，当补丁无法启用时，错误代码可能会被用来拒绝加载模块.

下面演示一个Patch示例(不建议在主机上尝试，可以在虚拟机上尝试，可能造成崩溃).

```c
// livepatch_example.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/livepatch.h>

// 新函数的实现
static asmlinkage long my_sys_read(unsigned int fd, char __user *buf, size_t count)
{
    pr_info("my_sys_read: fd=%u, buf=%p, count=%zu\n", fd, buf, count);
    // 调用原始函数（通过 Livepatch 自动处理）
    return ((long (*)(unsigned int, char __user *, size_t))klp_get_original_func("sys_read"))(fd, buf, count);
}

// 定义 klp_func：描述要替换的函数
static struct klp_func funcs[] = {
    {
        .old_name = "sys_read",           // 旧函数名
        .new_func = (void *)my_sys_read,  // 新函数地址
    },
    { } // 数组必须以空结构体结尾
};

// 定义 klp_object：绑定到内核本身（vmlinux）
static struct klp_object objs[] = {
    {
        .name = NULL,          // 内核本身（vmlinux）用 NULL 表示
        .funcs = funcs,        // 关联上面的 klp_func 数组
    },
    { } // 数组必须以空结构体结尾
};

// 定义 klp_patch：整个补丁的入口
static struct klp_patch my_patch = {
    .mod = THIS_MODULE,        // 指向当前模块
    .objs = objs,              // 关联上面的 klp_object 数组
};

// 模块初始化函数：加载时启用补丁
static int livepatch_init(void) // module_init函数
{
    int ret;

    // 启用补丁
    ret = klp_enable_patch(&my_patch);
    if (ret) {
        pr_err("Failed to enable patch: %d\n", ret);
        return ret;
    }

    pr_info("Livepatch module loaded, patch enabled!\n");
    return 0;
}

// 模块退出函数：卸载时自动恢复补丁
static void livepatch_exit(void)
{
    pr_info("Livepatch module unloaded, patch reverted!\n");
}

module_init(livepatch_init);
module_exit(livepatch_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Livepatch example: replace sys_read");
```

编译进内核.

```makefile
obj-m += livepatch_example.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

加载模块.

```bash
sudo insmod livepatch_example.ko
```

## Livepatch 回调机制(Callbacks)

利用回调函数在内核对象被补丁应用（patch）或取消补丁（unpatch）时，执行一些额外的操作. 该机制可以让 Livepatch 模块在补丁操作前后执行特定的回调函数.

1. Pre-patch

    在某个 klp_object 被补丁之前执行.

2. Post-patch

    在补丁应用后执行，补丁已经生效，并且所有任务都已经开始使用新的补丁代码.

3. Pre-unpatch

    在补丁被卸载之前执行，此时补丁代码仍然活跃.

4. Post-unpatch

    在补丁被卸载之后执行，此时补丁代码已经完全恢复，并且没有任务在运行补丁代码.

在使用这些回调时，通常需要配合 内存屏障（memory barriers）和 内核同步机制（如互斥锁、spinlock 或 stop_machine() 等）来避免并发问题.

## Livepatch 累积补丁(Cumulative Patches)

补丁之间可能存在依赖关系. 如果多个补丁需要对同一个函数进行不同的修改，那么我们就需要定义补丁的安装顺序. 这可能会成为维护工作的噩梦，尤其是当多个补丁以不同方式修改同一个函数时. 一种优雅的解决方案是使用 "Atomic Replace" 功能. 它允许创建所谓的 "Cumulative Patches"。 这些补丁包含所有旧版 Livepatches 中需要的更改，并在一次过渡中完全取代它们.

官方文档: https://www.kernel.org/doc/html/latest/livepatch/cumulative-patches.html

### 使用方式

通过 "replace" 使能.

```c
static struct klp_patch patch = {
        .mod = THIS_MODULE,
        .objs = objs,
        .replace = true,
};
```

## Livepatch 的 ELF 格式

livepatch 模块必须遵循的 ELF 格式要求. 以前，Livepatch 通过在补丁模块中嵌入 dynrela（动态重定位）部分来解决这个问题. 这种方法有缺点，因为它需要为每种架构编写特定的代码来处理这些重定位，导致代码重复且难以移植. 现在，Livepatch 使用 SHT_RELA 重定位部分来处理这些重定位，而不再使用旧的 dynrela 部分, 负责自己的重定位部分，并解析那些特定于 Livepatch 的符号（即补丁代码中使用的符号）.

### modinfo字段

Livepatch 模块必须具有 "livepatch "modinfo 属性. 用户可以通过使用 "modinfo "命令并查看是否存在 "livepatch "字段来识别 Livepatch 模块. 内核模块加载器也使用该字段来识别 livepatch 模块.

```bash
% modinfo livepatch-meminfo.ko
filename:               livepatch-meminfo.ko
livepatch:              Y
license:                GPL
depends:
vermagic:               4.3.0+ SMP mod_unload
```

### Livepatch relocation sections

Livepatch 重定位部分是 `SHT_RELA` 类型的部分，还带有 `SHF_RELA_LIVEPATCH` 标志. 重定位section的名称必须符合以下格式.

```bash
# 例子
# .klp.rela.ext4.text.ext4_attr_store
# .klp.rela.vmlinux.text.cmdline_proc_show

.klp.rela.objname.section_name
^        ^^     ^ ^          ^
|________||_____| |__________|
   [A]      [B]        [C]

[A]

The relocation section name is prefixed with the string “.klp.rela.”

[B]

The name of the object (i.e. “vmlinux” or name of module) to which the relocation section belongs follows immediately after the prefix.

[C]

The actual name of the section to which this relocation section applies.
```

### Livepatch symbol

symbol的 `st_shndx` 字段被设置为 `SHN_LIVEPATCH`.

```bash
# 例子
# .klp.sym.vmlinux.snprintf,0
# .klp.sym.vmlinux.printk,0
# .klp.sym.btrfs.btrfs_ktype,0

.klp.sym.objname.symbol_name,sympos
^       ^^     ^ ^         ^ ^
|_______||_____| |_________| |
   [A]     [B]       [C]    [D]

[A]

The symbol name is prefixed with the string “.klp.sym.”

[B]

The name of the object (i.e. “vmlinux” or name of module) to which the symbol belongs follows immediately after the prefix.

[C]

The actual name of the symbol.

[D]

The position of the symbol in the object (as according to kallsyms) This is used to differentiate duplicate symbols within the same object. The symbol position is expressed numerically (0, 1, 2...). The symbol position of a unique symbol is 0.
```

## Shadow 变量

`Shadow Variables` 的意义在于不修改现有数据结构的前提下，动态地为这些结构附加额外的数据. 可以在 `livepatch/shadow.c` 查看 APIs.

源码实现: https://elixir.bootlin.com/linux/v6.14-rc2/source/kernel/livepatch/shadow.c

### Datastructures

klp_shadow

```c
/**
 * struct klp_shadow - shadow variable structure
 * @node:	klp_shadow_hash hash table node
 * @rcu_head:	RCU is used to safely free this structure
 * @obj:	pointer to parent object
 * @id:		data identifier
 * @data:	data area
 */
struct klp_shadow {
	struct hlist_node node;
	struct rcu_head rcu_head;
	void *obj; // 指向父对象的指针（如 struct task_struct *）
	unsigned long id; // 用户定义的标识符（如 0 表示版本1，1 表示版本2）
	char data[]; // 实际存储的影子数据（可以是任意类型）
};
```

### APIs

`klp_shadow_get()` 获取一个已存在的 Shadow 变量的指针

```c
/**
 * klp_shadow_get() - retrieve a shadow variable data pointer
 * @obj:	pointer to parent object
 * @id:		data identifier
 *
 * Return: the shadow variable data element, NULL on failure.
 */
void *klp_shadow_get(void *obj, unsigned long id)
{
	struct klp_shadow *shadow;

	rcu_read_lock();

	hash_for_each_possible_rcu(klp_shadow_hash, shadow, node,
				   (unsigned long)obj) {

		if (klp_shadow_match(shadow, obj, id)) {
			rcu_read_unlock();
			return shadow->data;
		}
	}

	rcu_read_unlock();

	return NULL;
}
```

`klp_shadow_alloc()` 分配并添加一个新的 Shadow 变量

```c
/**
 * klp_shadow_alloc() - allocate and add a new shadow variable
 * @obj:	pointer to parent object
 * @id:		data identifier
 * @size:	size of attached data
 * @gfp_flags:	GFP mask for allocation
 * @ctor:	custom constructor to initialize the shadow data (optional)
 * @ctor_data:	pointer to any data needed by @ctor (optional)
 *
 * Allocates @size bytes for new shadow variable data using @gfp_flags.
 * The data are zeroed by default.  They are further initialized by @ctor
 * function if it is not NULL.  The new shadow variable is then added
 * to the global hashtable.
 *
 * If an existing <obj, id> shadow variable can be found, this routine will
 * issue a WARN, exit early and return NULL.
 *
 * This function guarantees that the constructor function is called only when
 * the variable did not exist before.  The cost is that @ctor is called
 * in atomic context under a spin lock.
 *
 * Return: the shadow variable data element, NULL on duplicate or
 * failure.
 */
void *klp_shadow_alloc(void *obj, unsigned long id,
		       size_t size, gfp_t gfp_flags,
		       klp_shadow_ctor_t ctor, void *ctor_data)
{
	return __klp_shadow_get_or_alloc(obj, id, size, gfp_flags,
					 ctor, ctor_data, true);
}
```

`klp_shadow_get_or_alloc()` 获取已存在的 Shadow 变量，如果不存在则分配一个新的 Shadow 变量

```c
/**
 * klp_shadow_get_or_alloc() - get existing or allocate a new shadow variable
 * @obj:	pointer to parent object
 * @id:		data identifier
 * @size:	size of attached data
 * @gfp_flags:	GFP mask for allocation
 * @ctor:	custom constructor to initialize the shadow data (optional)
 * @ctor_data:	pointer to any data needed by @ctor (optional)
 *
 * Returns a pointer to existing shadow data if an <obj, id> shadow
 * variable is already present.  Otherwise, it creates a new shadow
 * variable like klp_shadow_alloc().
 *
 * This function guarantees that only one shadow variable exists with the given
 * @id for the given @obj.  It also guarantees that the constructor function
 * will be called only when the variable did not exist before.  The cost is
 * that @ctor is called in atomic context under a spin lock.
 *
 * Return: the shadow variable data element, NULL on failure.
 */
void *klp_shadow_get_or_alloc(void *obj, unsigned long id,
			      size_t size, gfp_t gfp_flags,
			      klp_shadow_ctor_t ctor, void *ctor_data)
{
	return __klp_shadow_get_or_alloc(obj, id, size, gfp_flags,
					 ctor, ctor_data, false);
}
EXPORT_SYMBOL_GPL(klp_shadow_get_or_alloc);

static void klp_shadow_free_struct(struct klp_shadow *shadow,
				   klp_shadow_dtor_t dtor)
{
	hash_del_rcu(&shadow->node);
	if (dtor)
		dtor(shadow->obj, shadow->data);
	kfree_rcu(shadow, rcu_head);
}
```

`klp_shadow_free()` 释放一个指定的 Shadow 变量并从哈希表中移除

```c
/**
 * klp_shadow_free() - detach and free a <obj, id> shadow variable
 * @obj:	pointer to parent object
 * @id:		data identifier
 * @dtor:	custom callback that can be used to unregister the variable
 *		and/or free data that the shadow variable points to (optional)
 *
 * This function releases the memory for this <obj, id> shadow variable
 * instance, callers should stop referencing it accordingly.
 */
void klp_shadow_free(void *obj, unsigned long id, klp_shadow_dtor_t dtor)
{
	struct klp_shadow *shadow;
	unsigned long flags;

	spin_lock_irqsave(&klp_shadow_lock, flags);

	/* Delete <obj, id> from hash */
	hash_for_each_possible(klp_shadow_hash, shadow, node,
			       (unsigned long)obj) {

		if (klp_shadow_match(shadow, obj, id)) {
			klp_shadow_free_struct(shadow, dtor);
			break;
		}
	}

	spin_unlock_irqrestore(&klp_shadow_lock, flags);
}
```

`klp_shadow_free_all()` 释放所有 Shadow 变量并从哈希表中移除

```c
/**
 * klp_shadow_free_all() - detach and free all <_, id> shadow variables
 * @id:		data identifier
 * @dtor:	custom callback that can be used to unregister the variable
 *		and/or free data that the shadow variable points to (optional)
 *
 * This function releases the memory for all <_, id> shadow variable
 * instances, callers should stop referencing them accordingly.
 */
void klp_shadow_free_all(unsigned long id, klp_shadow_dtor_t dtor)
{
	struct klp_shadow *shadow;
	unsigned long flags;
	int i;

	spin_lock_irqsave(&klp_shadow_lock, flags);

	/* Delete all <_, id> from hash */
	hash_for_each(klp_shadow_hash, i, shadow, node) {
		if (klp_shadow_match(shadow, shadow->obj, id))
			klp_shadow_free_struct(shadow, dtor);
	}

	spin_unlock_irqrestore(&klp_shadow_lock, flags);
}
```

## 可靠栈追踪(Reliable Stacktrace)

内核的 Livepatch 一致性模型 依赖于准确识别可能具有活跃状态的函数.

### 移植的要求

不同架构必须实现一个可靠的堆栈追踪函数：

使用 `CONFIG_ARCH_STACKWALK` 的架构必须实现 `arch_stack_walk_reliable()`。
其他架构则必须实现 `save_stack_trace_tsk_reliable()`.

该函数的功能是: 如果堆栈追踪是可靠的，返回零，表示追踪没有遗漏任何函数. 如果堆栈追踪不可靠，返回非零值，表示该追踪不可靠.

为了确保内核代码能够在所有情况下正确地进行回溯，架构可能需要验证代码是否以 unwinder（回溯器）期望的方式进行编译。例如，回溯器可能要求函数以特定的方式操作堆栈指针，或者要求所有函数使用特定的函数前后操作。架构可以使用 `objtool` 工具来验证编译是否符合这些要求。

## 移植思路

https://www.kernel.org/doc/html/latest/livepatch/livepatch.html#id6

要为新架构添加一致性模型支持，有以下几种选择：添加 `CONFIG_HAVE_RELIABLE_STACKTRACE`. 这意味着要移植 `objtool`，对于非DWARF 解调器，还要确保堆栈跟踪代码有办法检测堆栈上的中断. 或者，确保每个 kthread 都在安全位置调用 `klp_update_patch_state()`. Kthreads 通常处于重复执行某些操作的无限循环中. 切换 kthread 补丁状态的安全位置是循环中的指定点，在这里没有锁，所有数据结构都处于定义明确的状态. 当使用工作队列或 kthread Worker API 时，这个位置就很清楚了，这些 kthread 在通用循环中处理独立的操作. 这些 kthreads 在一个通用循环中处理独立的操作，而具有自定义循环的 kthreads 则要复杂得多. 在这种情况下，没有 `HAVE_RELIABLE_STACKTRACE` 的 arches 仍然可以使用一致性模型的非堆栈检查部分：在用户任务跨越内核/用户空间边界时对其进行修补；在指定的修补点对 kthreads 和空闲任务进行修补. 这个方案不如方案 1 好，因为它需要向用户任务发出信号，并唤醒 kthreads 对其进行修补. 但对于那些还没有可靠堆栈跟踪的架构来说，这仍然是一个不错的备用方案.

## References

1. laokz老师的文档: https://gitee.com/laokz/oerv/blob/master/livepatch/livepatch.md

2. livepatch 官方说明: https://www.kernel.org/doc/html/latest/livepatch/livepatch.html#consistency-model

3. livepatch的ELF格式: https://www.kernel.org/doc/html/latest/livepatch/module-elf-format.html

4. Livepatching APIs: https://www.kernel.org/doc/html/latest/livepatch/api.html