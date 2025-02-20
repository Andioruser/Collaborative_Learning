## 简介

ORC 数据包括由 objtool 生成的 unwind tables. 在分析 .o 文件的所有代码路径后，它会确定文件中每个指令地址的堆栈状态信息，并将该信息输出到 `.orc_unwind` 和 `.orc_unwind_ip` 部分.

为了从内核中实际获取 ORC 信息，我们首先需要编译一个包含 ORC 信息的内核. 在上游，x86_64 的默认情况就是这样. 我们需要确保在配置源代码树时使用 `CONFIG_UNWINDER_ORC=y`，还需要禁用 `CONFIG_UNWINDER_FRAME_POINTER` 和 `CONFIG_FRAME_POINTER`，以摆脱帧指针.

### Unwind method

1. Frame Pointers(栈帧指针): 每当函数被调用时，栈帧指针会占用一个CPU寄存器，而这个寄存器实际上在大多数时间里并不需要. 每次函数调用时，栈帧指针需要被推送到栈上，函数返回时再从栈中弹出. 这会增加不必要的指令，消耗宝贵的寄存器资源.

2. [DWARF](DWARF.md): DWARF格式包含了程序中调试信息的元数据，包括变量名、函数名、行号等等. 这些信息可以帮助开发人员在程序出现问题时进行调试和分析。DWARF格式通常与编译器一起使用，编译器会将调试信息嵌入到生成的可执行文件中。在DWARF标准中，定义了一种名为CFI(Call Frame Information)的标准，用来帮助我们进行栈回溯.

3. ORC: ORC 是一种简化的调试信息格式，它只包含展开栈所需的信息. 
    - 相比于DWARF调试信息，ORC只关注栈展开所需的数据，它摆脱了复杂的 DWARF CFI 状态机，也摆脱了对不必要寄存器的跟踪.
    - 相比栈帧指针，ORC不需要占用额外的CPU寄存器或内存空间，因此可以减少开销.

## 使用 ORC 来实现 unwind

ORC的文本格式示例如下:

```bash
.text+325d: sp:(und) bp:(und) type:call end:0
.text+3260: sp:sp+8 bp:(und) type:call end:0
.text+3262: sp:sp+16 bp:(und) type:call end:0
.text+3267: sp:sp+24 bp:(und) type:call end:0
.text+326b: sp:sp+32 bp:prevsp-32 type:call end:0
.text+326f: sp:sp+40 bp:prevsp-32 type:call end:0
.text+3273: sp:sp+96 bp:prevsp-32 type:call end:0
.text+331d: sp:sp+40 bp:prevsp-32 type:call end:0
.text+331e: sp:sp+32 bp:prevsp-32 type:call end:0
.text+331f: sp:sp+24 bp:(und) type:call end:0
.text+3321: sp:sp+16 bp:(und) type:call end:0
.text+3323: sp:sp+8 bp:(und) type:call end:0

# sp：表示如何计算之前的栈指针 (PREV_RSP).
# bp：表示如何计算之前的帧指针 (PREV_RBP).
# type:call：指明这是一种函数调用类型的栈信息.
```

首先我们理清一些概念.

- `PREV_RSP`： 由 sp 字段计算得出. sp 字段提供了栈指针的计算方法，通常是通过某个寄存器加上一个偏移量来得到.

- `PREV_RBP`： 由 bp 字段计算。对于没有作为栈帧指针使用的 RBP，x86_64 ABI 要求它在函数调用中保存，因此我们需要从栈中读取 RBP 的值. 如果 bp 字段写成 bp:prevsp-32，那么我们就从 PREV_RSP - 32 的栈位置读取值来计算 PREV_RBP.

- `PREV_RIP`： 由 PREV_RSP 计算. 由于在正常的函数调用中，RIP 被推送到栈中，通常存储在 RSP - 8 的位置. 因此，可以通过读取栈上 PREV_RSP - 8 位置的值来得到 PREV_RIP.

### ORC中的 unwind 算法

1. 查找对应的 ORC 记录： 每条记录都与一个指令偏移量关联，且这些记录按顺序排列。通过指令偏移量，可以查找到对应的 ORC 记录.

2. 计算 `PREV_RSP`（前一个栈指针）： 根据 sp 字段，计算出栈指针 PREV_RSP，这通常是将一个寄存器值加上一个偏移量.

3. 计算 `PREV_RIP`（前一个指令指针）： 由于 RIP 通常在函数调用时被推送到栈中，我们可以通过读取栈上 PREV_RSP - 8 位置的值来计算 PREV_RIP.

4. 计算 `PREV_RBP`（前一个帧指针）： 根据 bp 字段，计算 PREV_RBP，有时它是从栈中读取的值，或者如果 bp 字段未定义，则表示 RBP 在当前函数调用中保持不变.

> It may seem a bit odd that we’re computing the previous value of RBP. It turns out that some stack frames still use frame pointers. There are likely several reasons for this. One may be that they are useful for the compiler when a function has many local variables. But another crucial reason is that the kernel just-in-time compiles code (for instance, eBPF code) which uses frame pointers. It would be a pain for these just-in-time compilers to also emit ORC entries for these functions.

### 示例

假设当前

- RIP: ffffffffae2f1da3
- RSP: ffffa764404a3bb8
- RBP: 0000000000000002

1. 条目如下，我们的RIP对应的偏移量是`.text+0x2f1da3`. 因为 0x2f1da3 落在 0x2f1d03 和 0x2f1d10 之间，所以我们会选择 .text+2f1d03 作为匹配的记录.

    ```bash
    .text+2f1d03: sp:sp+80 bp:prevsp-48 type:call end:0
    .text+2f1d10: sp:sp+64 bp:prevsp-32 type:call end:0
    ```

2. 计算 `PREV_RSP`

    ```bsah
    PREV_RSP = RSP + 80
         = 0xffffa764404a3bb8 + 80
         = 0xffffa764404a3c08
    ```

3. 计算 `PREV_RIP`

    ```bash
    PREV_RIP = *(PREV_RSP - 8)
         = *(0xffffa764404a3c08 - 8)
         = *(0xffffa764404a3c00)
         = 0xffffffffae2e07d2
    ```

4. 计算 `PREV_RBP`

    ```bash
    # 计算得出 PREV_RBP = 0x0000000000000000，表示上一层栈帧中没有有效的帧指针，或者 RBP 没有被推送到栈上

    PREV_RBP = *(PREV_RSP - 48)
         = *(0xffffa764404a3c08 - 48)
         = *(0xffffa764404a3bd8)
         = 0x0000000000000000
    ```

需要注意的是，如果如果遇到条目为 `und` 例如 `bp:(und)` 则 PREV_RBP = RBP.

> frame pointer was not pushed to the stack or changed since the previous stack frame.

当遇到 `type:regs`，这表明此栈帧对应的是一个寄存器转储（pt_regs），即内核保存的用户空间寄存器数据. 此时，堆栈展开已经完成，因为这是内核保存寄存器的地方，表示栈展开已经完成.

## References

1. linux doc: https://docs.kernel.org/arch/x86/orc-unwinder.html

2. laokz 老师推荐的 oracle blogs: https://blogs.oracle.com/linux/post/unwinding-stack-frame-pointers-and-orc