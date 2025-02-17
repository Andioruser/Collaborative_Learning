## 加载器

```c
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>
#include <elf.h>

// RISC-V 64位ABI相关定义
#define RISCV_PAGE_SIZE 4096
#define STACK_SIZE      (4 * 1024 * 1024)  // 4MB栈空间

// ELF文件头验证
int verify_elf_header(Elf64_Ehdr *ehdr) {
    if (memcmp(ehdr->e_ident, ELFMAG, SELFMAG) != 0) {
        fprintf(stderr, "Not an ELF file\n");
        return -1;
    }
    if (ehdr->e_ident[EI_CLASS] != ELFCLASS64) {
        fprintf(stderr, "Not a 64-bit ELF\n");
        return -1;
    }
    if (ehdr->e_ident[EI_DATA] != ELFDATA2LSB) {
        fprintf(stderr, "Not little-endian\n");
        return -1;
    }
    if (ehdr->e_machine != EM_RISCV) {
        fprintf(stderr, "Not a RISC-V executable\n");
        return -1;
    }
    return 0;
}

// 加载程序段到内存
void load_segments(int fd, Elf64_Phdr *phdrs, int phnum) {
    for (int i = 0; i < phnum; i++) {
        Elf64_Phdr *ph = &phdrs[i];

        if (ph->p_type != PT_LOAD) continue;

        // 计算对齐参数
        size_t offset = ph->p_offset & ~(RISCV_PAGE_SIZE - 1);
        size_t delta = ph->p_offset - offset;
        size_t mem_size = (ph->p_memsz + delta + RISCV_PAGE_SIZE - 1) & ~(RISCV_PAGE_SIZE - 1);

        // 映射内存区域
        void *addr = mmap(
            (void*)(ph->p_vaddr - delta),
            mem_size,
            PROT_READ | PROT_WRITE | PROT_EXEC,
            MAP_PRIVATE,
            fd,
            offset
        );

        if (addr == MAP_FAILED) {
            perror("mmap failed");
            exit(EXIT_FAILURE);
        }

        // 清零.bss区域
        if (ph->p_filesz < ph->p_memsz) {
            memset((char*)addr + ph->p_filesz, 0, ph->p_memsz - ph->p_filesz);
        }
    }
}

// 设置初始栈内容（遵循RISC-V ABI）
void* setup_stack(uint64_t *stack_top, int argc, char **argv, char **envp) {
    // 计算参数总长度
    size_t arg_size = 0;
    for (int i = 0; i < argc; i++) 
        arg_size += strlen(argv[i]) + 1;

    // 对齐栈指针（RISC-V要求16字节对齐）
    uint64_t *sp = stack_top;
    sp = (uint64_t*)((uintptr_t)sp - arg_size);
    sp = (uint64_t*)((uintptr_t)sp & ~0xF);

    // 复制参数到栈
    char *arg_str = (char*)sp;
    for (int i = 0; i < argc; i++) {
        strcpy(arg_str, argv[i]);
        arg_str += strlen(argv[i]) + 1;
    }

    // 构建argv指针数组
    uint64_t *argv_ptrs = (uint64_t*)sp - (argc + 1);
    argv_ptrs[argc] = 0;  // NULL结尾
    for (int i = argc-1; i >= 0; i--) {
        argv_ptrs[i] = (uint64_t)(arg_str - (argc - i) * (strlen(argv[i]) + 1));
    }

    // 设置寄存器值（通过伪造栈帧）
    struct {
        uint64_t argc;
        uint64_t argv;
        uint64_t envp;
        uint64_t auxv[32]; // 辅助向量
    } *stack_frame = (void*)argv_ptrs - sizeof(*stack_frame);

    stack_frame->argc = argc;
    stack_frame->argv = (uint64_t)argv_ptrs;
    stack_frame->envp = (uint64_t)envp;

    // 设置辅助向量（AT_NULL结尾）
    int auxv_idx = 0;
    stack_frame->auxv[auxv_idx++] = AT_PHDR;
    stack_frame->auxv[auxv_idx++] = 1; /* 实际phdr地址 */
    stack_frame->auxv[auxv_idx++] = AT_PHENT;
    stack_frame->auxv[auxv_idx++] = sizeof(Elf64_Phdr);
    stack_frame->auxv[auxv_idx++] = AT_ENTRY;
    stack_frame->auxv[auxv_idx++] = 1; /* 入口地址 */
    stack_frame->auxv[auxv_idx] = AT_NULL;

    return stack_frame;
}

// 程序入口跳转（RISC-V汇编实现）
__attribute__((naked)) 
void jump_to_entry(uint64_t entry, void *stack) {
    __asm__ volatile (
        "mv sp, %[stack]\n"   // 设置栈指针
        "li a0, 0\n"          // argc
        "mv a1, sp\n"         // argv
        "jalr %[entry]\n"     // 跳转到入口地址
        "li a7, 93\n"         // exit系统调用号
        "ecall\n"             // 退出
        : 
        : [entry] "r" (entry), [stack] "r" (stack)
        : "memory"
    );
}

int main(int argc, char **argv, char **envp) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <elf-executable>\n", argv[0]);
        return EXIT_FAILURE;
    }

    // 打开ELF文件
    int fd = open(argv[1], O_RDONLY);
    if (fd == -1) {
        perror("open failed");
        return EXIT_FAILURE;
    }

    // 读取ELF头
    Elf64_Ehdr ehdr;
    if (read(fd, &ehdr, sizeof(ehdr)) != sizeof(ehdr)) {
        perror("read ELF header failed");
        close(fd);
        return EXIT_FAILURE;
    }

    // 验证ELF头
    if (verify_elf_header(&ehdr) != 0) {
        close(fd);
        return EXIT_FAILURE;
    }

    // 读取程序头表
    Elf64_Phdr *phdrs = malloc(ehdr.e_phentsize * ehdr.e_phnum);
    lseek(fd, ehdr.e_phoff, SEEK_SET);
    if (read(fd, phdrs, ehdr.e_phentsize * ehdr.e_phnum) != ehdr.e_phentsize * ehdr.e_phnum) {
        perror("read program headers failed");
        free(phdrs);
        close(fd);
        return EXIT_FAILURE;
    }

    // 加载所有LOAD段
    load_segments(fd, phdrs, ehdr.e_phnum);

    // 分配栈空间
    void *stack = mmap(NULL, STACK_SIZE, PROT_READ | PROT_WRITE, 
                      MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
    if (stack == MAP_FAILED) {
        perror("stack allocation failed");
        free(phdrs);
        close(fd);
        return EXIT_FAILURE;
    }

    uint64_t *stack_top = (uint64_t*)((char*)stack + STACK_SIZE);

    // 设置栈内容
    void *initial_sp = setup_stack(stack_top, argc-1, argv+1, envp);

    // 准备跳转到入口点
    jump_to_entry(ehdr.e_entry, initial_sp);

    // 永远不会执行到这里
    free(phdrs);
    close(fd);
    return EXIT_SUCCESS;
}
```