## 简介

livepatch patching functions.

## Datastructures

klp_ops

```c
/**
 * struct klp_ops - structure for tracking registered ftrace ops structs
 *
 * A single ftrace_ops is shared between all enabled replacement functions
 * (klp_func structs) which have the same old_func.  This allows the switch
 * between function versions to happen instantaneously by updating the klp_ops
 * struct's func_stack list.  The winner is the klp_func at the top of the
 * func_stack (front of the list).
 *
 * @node:	node for the global klp_ops list
 * @func_stack:	list head for the stack of klp_func's (active func is on top)
 * @fops:	registered ftrace ops struct
 */
struct klp_ops {
	struct list_head node;
	struct list_head func_stack;
	struct ftrace_ops fops; // 注册到 ftrace 的操作结构，用于拦截原函数调用并跳转到补丁函数
};
```

## patch.c

### 流程解析

```bash
                        加载补丁模块
                        │
                        ▼
                        klp_init_patch_early (初始化结构)
                        │
                        ▼
                        klp_init_patch (注册到sysfs)
                        │
                        ▼
                        klp_enable_patch (启用补丁)
                        │
                        ▼
                        klp_patch_object (遍历对象)
                        │
                        ▼
                        klp_patch_func (应用单函数补丁)
                        │
                        ▼
                        注册ftrace_handler (重定向执行流)
                        │
                        ▼
                        函数调用触发 klp_ftrace_handler
                        │
                        ▼
                        跳转到补丁函数 new_func
```

## 其他函数

```c
static LIST_HEAD(klp_ops);

/**
 * 根据原函数地址查找对应的 klp_ops 结构
 * @old_func: 原函数的地址
 * @return: 找到的 klp_ops 结构指针，未找到返回 NULL
 */
struct klp_ops *klp_find_ops(void *old_func)
{
	struct klp_ops *ops;
	struct klp_func *func;

	list_for_each_entry(ops, &klp_ops, node) {
		func = list_first_entry(&ops->func_stack, struct klp_func,
					stack_node);

        // 检查栈顶函数的原地址是否匹配
		if (func->old_func == old_func)
			return ops;
	}

	return NULL;
}

/**
 * Ftrace 回调函数，拦截原函数调用并跳转到补丁函数
 * @ip: 原函数的指令地址
 * @parent_ip: 调用者地址（通常未使用）
 * @fops: 关联的 ftrace_ops 结构
 * @fregs: 寄存器状态，用于修改指令指针
 */
static void notrace klp_ftrace_handler(unsigned long ip,
				       unsigned long parent_ip,
				       struct ftrace_ops *fops,
				       struct ftrace_regs *fregs)
{
	struct klp_ops *ops;
	struct klp_func *func;
	int patch_state;
	int bit;

	ops = container_of(fops, struct klp_ops, fops);

	/*
	 * The ftrace_test_recursion_trylock() will disable preemption,
	 * which is required for the variant of synchronize_rcu() that is
	 * used to allow patching functions where RCU is not watching.
	 * See klp_synchronize_transition() for more details.
	 */
	bit = ftrace_test_recursion_trylock(ip, parent_ip);
	if (WARN_ON_ONCE(bit < 0))
		return;

    // 获取当前激活的补丁函数（栈顶元素）
	func = list_first_or_null_rcu(&ops->func_stack, struct klp_func,
				      stack_node);

	/*
	 * func should never be NULL because preemption should be disabled here
	 * and unregister_ftrace_function() does the equivalent of a
	 * synchronize_rcu() before the func_stack removal.
	 */
	if (WARN_ON_ONCE(!func))
		goto unlock;

	/*
	 * In the enable path, enforce the order of the ops->func_stack and
	 * func->transition reads.  The corresponding write barrier is in
	 * __klp_enable_patch().
	 *
	 * (Note that this barrier technically isn't needed in the disable
	 * path.  In the rare case where klp_update_patch_state() runs before
	 * this handler, its TIF_PATCH_PENDING read and this func->transition
	 * read need to be ordered.  But klp_update_patch_state() already
	 * enforces that.)
	 */
	smp_rmb();

	if (unlikely(func->transition)) {

		/*
		 * Enforce the order of the func->transition and
		 * current->patch_state reads.  Otherwise we could read an
		 * out-of-date task state and pick the wrong function.  The
		 * corresponding write barrier is in klp_init_transition().
		 */
		smp_rmb();

		patch_state = current->patch_state;

		WARN_ON_ONCE(patch_state == KLP_TRANSITION_IDLE);

		if (patch_state == KLP_TRANSITION_UNPATCHED) {
			/*
			 * Use the previously patched version of the function.
			 * If no previous patches exist, continue with the
			 * original function.
			 */
			func = list_entry_rcu(func->stack_node.next,
					      struct klp_func, stack_node);

			if (&func->stack_node == &ops->func_stack)
				goto unlock;
		}
	}

	/*
	 * NOPs are used to replace existing patches with original code.
	 * Do nothing! Setting pc would cause an infinite loop.
	 */
	if (func->nop)
		goto unlock;

    // 跳转到补丁函数
	ftrace_regs_set_instruction_pointer(fregs, (unsigned long)func->new_func);

unlock:
	ftrace_test_recursion_unlock(bit);
}

/**
 * 撤销单个函数的补丁，恢复原函数
 * @func: 要撤销的补丁函数结构
 */
static void klp_unpatch_func(struct klp_func *func)
{
	struct klp_ops *ops;

	if (WARN_ON(!func->patched))
		return;
	if (WARN_ON(!func->old_func))
		return;

	ops = klp_find_ops(func->old_func);
	if (WARN_ON(!ops))
		return;

	if (list_is_singular(&ops->func_stack)) {
		unsigned long ftrace_loc;

		ftrace_loc = ftrace_location((unsigned long)func->old_func);
		if (WARN_ON(!ftrace_loc))
			return;

        // 注销 ftrace 回调并移除过滤器
		WARN_ON(unregister_ftrace_function(&ops->fops));
		WARN_ON(ftrace_set_filter_ip(&ops->fops, ftrace_loc, 1, 0));

        // 从全局链表移除 ops 并释放内存
		list_del_rcu(&func->stack_node);
		list_del(&ops->node);
		kfree(ops);
	} else {
		list_del_rcu(&func->stack_node);
	}

	func->patched = false;
}

/**
 * 应用单个函数的补丁，替换原函数
 * @func: 要应用的补丁函数结构
 * @return: 成功返回 0，失败返回错误码
 */
static int klp_patch_func(struct klp_func *func)
{
	struct klp_ops *ops;
	int ret;

	if (WARN_ON(!func->old_func))
		return -EINVAL;

	if (WARN_ON(func->patched))
		return -EINVAL;

	ops = klp_find_ops(func->old_func);
	if (!ops) {
		unsigned long ftrace_loc;

		ftrace_loc = ftrace_location((unsigned long)func->old_func);
		if (!ftrace_loc) {
			pr_err("failed to find location for function '%s'\n",
				func->old_name);
			return -EINVAL;
		}

		ops = kzalloc(sizeof(*ops), GFP_KERNEL);
		if (!ops)
			return -ENOMEM;

        // 初始化 ftrace_ops
		ops->fops.func = klp_ftrace_handler;
		ops->fops.flags = FTRACE_OPS_FL_DYNAMIC |
#ifndef CONFIG_HAVE_DYNAMIC_FTRACE_WITH_ARGS
				  FTRACE_OPS_FL_SAVE_REGS |
#endif
				  FTRACE_OPS_FL_IPMODIFY |
				  FTRACE_OPS_FL_PERMANENT;

		list_add(&ops->node, &klp_ops);

		INIT_LIST_HEAD(&ops->func_stack);
		list_add_rcu(&func->stack_node, &ops->func_stack);

        // 设置 ftrace 过滤器（仅拦截目标函数）
		ret = ftrace_set_filter_ip(&ops->fops, ftrace_loc, 0, 0);
		if (ret) {
			pr_err("failed to set ftrace filter for function '%s' (%d)\n",
			       func->old_name, ret);
			goto err;
		}

        // 注册 ftrace 回调
		ret = register_ftrace_function(&ops->fops);
		if (ret) {
			pr_err("failed to register ftrace handler for function '%s' (%d)\n",
			       func->old_name, ret);
			ftrace_set_filter_ip(&ops->fops, ftrace_loc, 1, 0);
			goto err;
		}


	} else {
		list_add_rcu(&func->stack_node, &ops->func_stack);
	}

	func->patched = true;

	return 0;

err:
	list_del_rcu(&func->stack_node);
	list_del(&ops->node);
	kfree(ops);
	return ret;
}

/**
 * 内部函数：撤销对象的所有或部分补丁
 * @obj: 目标对象
 * @nops_only: 是否仅撤销 NOP 类型的补丁
 */
static void __klp_unpatch_object(struct klp_object *obj, bool nops_only)
{
	struct klp_func *func;

	klp_for_each_func(obj, func) {
		if (nops_only && !func->nop)
			continue;
        
        // 根据 nops_only 决定是否跳过非 NOP 补丁
		if (func->patched)
			klp_unpatch_func(func);
	}

    // 更新对象补丁状态
	if (obj->dynamic || !nops_only)
		obj->patched = false;
}


void klp_unpatch_object(struct klp_object *obj)
{
	__klp_unpatch_object(obj, false);
}

/**
 * 应用对象的所有补丁
 * @obj: 目标对象
 * @return: 成功返回 0，失败回滚并返回错误码
 */
int klp_patch_object(struct klp_object *obj)
{
	struct klp_func *func;
	int ret;

	if (WARN_ON(obj->patched))
		return -EINVAL;

	klp_for_each_func(obj, func) {
		ret = klp_patch_func(func);
		if (ret) {
			klp_unpatch_object(obj);
			return ret;
		}
	}
	obj->patched = true; // 标记对象为已补丁

	return 0;
}

/**
 * 批量撤销补丁中所有对象的补丁
 * @patch: 目标补丁
 * @nops_only: 是否仅处理 NOP 补丁
 */
static void __klp_unpatch_objects(struct klp_patch *patch, bool nops_only)
{
	struct klp_object *obj;

	klp_for_each_object(patch, obj)
		if (obj->patched)
			__klp_unpatch_object(obj, nops_only);
}

void klp_unpatch_objects(struct klp_patch *patch)
{
	__klp_unpatch_objects(patch, false);
}

void klp_unpatch_objects_dynamic(struct klp_patch *patch)
{
	__klp_unpatch_objects(patch, true);
}
```