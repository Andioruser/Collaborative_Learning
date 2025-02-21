## 简介

Ptrace 可以让父进程控制子进程运行，并可以检查和改变子进程的核心image的功能（Peek and poke 在系统编程中是很知名的叫法，指的是直接读写内存内容）。ptrace主要跟踪的是进程运行时的状态，直到收到一个终止信号结束进程，这里的信号如果是我们给程序设置的断点，则进程被中止，并且通知其父进程，在进程中止的状态下，进程的内存空间可以被读写。当然父进程还可以使子进程继续执行，并选择是否忽略引起中止的信号，ptrace可以让一个进程监视和控制另一个进程的执行,并且修改被监视进程的内存、寄存器等,主要应用于断点调试和系统调用跟踪，strace和gdb工具就是基于ptrace编写的！

### 原理

在Linux系统中, 进程有一个状态是 TASK_TRACED, 当使用了ptrace跟踪后，所有发送给被跟踪的子进程的信号(除了SIGKILL)，都会被转发给父进程，而子进程则会被阻塞，这时子进程的状态就会被系统标注为TASK_TRACED，而父进程收到信号后，就可以对停止下来的子进程进行检查和修改，然后让子进程继续运行.

## 使用方式

### 函数定义

详情请参看man手册 https://man7.org/linux/man-pages/man2/ptrace.2.html

- long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);

    - request: 表示要执行的操作类型, 反调试会用到PT_DENY_ATTACH，调试会用到PTRACE_ATTACH.详细
    - pid: 要操作的目标进程ID
    - addr: 要监控的目标内存地址
    - data: 保存读取出或者要写入的数据（根据request的参数而变化）.
    
- request详细参数

    - PTRACE_TRACEME：tracee表明自己想要被追踪，这会自动与父进程建立追踪关系，这也是唯一能被tracee使用的request，其他的request都由tracer指定.
    - PTRACE_ATTACH：tracer用来附着一个进程tracee，以建立追踪关系，并向其发送SIGSTOP信号使其暂停.
    - PTRACE_SEIZE：像PTRACE_ATTACH附着进程，但它不会让tracee暂停，addr参数须为0，data参数指定一位ptrace选项.
    - PTRACE_DETACH：解除追踪关系，tracee将继续运行.
    - PTRACE_GETREGS: 获取进程的寄存器数值.
    - PTRACE_SETREGS: 设置目标进程的寄存器.
    - PTRACE_GETFPREGS: 获取目标进程的浮点寄存器.
    - PTRACE_SETFPREGS: 设置目标进程的浮点寄存器.
    - PTRACE_SYSCALL: 让进程在系统调用入口和退出时停止.
    - PTRACE_CONT: 命令用来继续执行被调试的进程。它会将被暂停的进程（通常是因为调试器设置了断点）恢复运行.
    - PTRACE_SINGLESTEP: 单步调试是一个比较有趣的功能，当把被调试进程设置为单步调试模式后，被调试进程没执行一条CPU指令都会停止执行.
    - PTRACE_PEEKDATA: 读取目标进程内存中的数据, 不阻止子进程执行.
    - PTRACE_POKEDATA: 修改目标进程内存中的数据.
    - PTRACE_PEEKUSER: 读取目标进程用户寄存器.
    - PTRACE_POKEUSER: 修改目标进程用户寄存器.

## References

1. https://www.anquanke.com/post/id/231078

2. https://cloud.tencent.com/developer/article/1742878