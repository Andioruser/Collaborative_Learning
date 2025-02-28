# EasyCapture-rs    

easycapture-rs  

**linux 下的 TLS Master Key Log 工具**  

TLS Master Key Log工具是一种用于在Linux环境下记录和输出TLS（Transport Layer Security，传输层安全协议）通信过程中生成的主密钥的工具。    

这些主密钥在TLS握手阶段生成，用于保护后续的数据交换。     

这种工具通常被网络安全研究人员、性能测试人员以及需要分析或调试TLS加密流量的其他技术人员所使用。     

通过记录TLS会话中的主密钥，他们可以在事后对捕获的数据包进行解密，以便于进行进一步的安全研究或故障排查。 

一个常见的实现是通过设置【SSLKEYLOGFILE】环境变量来指定一个文件路径，TLS库（如OpenSSL）会在该文件中记录相关的密钥信息。     

之后，可以使用支持该功能的网络分析工具（例如Wireshark）加载这个日志文件以解密TLS流量。  

TLS密钥日志

TLS密钥日志是一种机制，允许解密TLS连接的流量。这主要通过将密钥信息记录到文件中，然后由外部程序（如Wireshark）使用这些文件来解密流量。

Wireshark   

Wireshark是一个广泛使用的网络协议分析器，它可以解密TLS流量。要使用Wireshark解密TLS流量，你需要提供一个密钥日志文件，其中包含用于建立加密连接的预主密钥。这可以通过Wireshark的首选项设置来完成，或者通过环境变量SSLKEYLOGFILE来配置。

环境变量SSLKEYLOGFILE   

SSLKEYLOGFILE是一个环境变量，可以用来指定Wireshark用于解密TLS流量的密钥日志文件的路径。在Linux系统上，你可以通过设置这个环境变量并启动浏览器来启用密钥日志记录。例如，你可以在终端中运行以下命令来设置环境变量并启动Chrome浏览器：  

```shell
export SSLKEYLOGFILE=/path/to/ssl.log   
google-chrome-stable
```

这样，当浏览器与服务器建立TLS连接时，它会将预主密钥记录到指定的日志文件中，Wireshark可以使用这个文件来解密后续的TLS流量。

如果你正在寻找在Linux系统上解密TLS流量的具体工具或方法，Wireshark结合SSLKEYLOGFILE环境变量是一个常用的解决方案。请注意，这些工具和方法的使用可能需要一定的技术知识，以及对TLS协议的基本理解。在使用这些工具时，请确保你了解它们的用途和潜在的安全风险。

目录结构    
├── allocator 内存分配器    
├── easycapture-ebpf ebpf 库    
└── easycapture 主程序  

easycapture-ebpf    

**这段代码是一个基于Rust的异步程序，主要用于通过eBPF技术监控OpenSSL的TLS连接，捕获关键加密参数。以下是对代码的逐步解析**    

1. 程序初始化
```rust
async fn main() -> ExitCode {
    // 设置日志级别（DEBUG/INFO）
    let level = if env::var("EASYCAPTURE_DEBUG").is_ok() {
        Level::DEBUG
    } else {
        Level::INFO
    };
    // 初始化tracing日志订阅者
    let subscriber = FmtSubscriber::builder().with_max_level(level).finish();
    tracing::subscriber::set_global_default(subscriber).unwrap();
    env_logger::init();
    //................
}
```
日志配置：根据环境变量EASYCAPTURE_DEBUG设置日志级别，使用tracing和env_logger进行日志记录。  

2.  资源清理    

```rust
defer! {
        if Path::new("/sys/fs/bpf/cloudwalker").exists() {
            _ = std::fs::remove_dir_all("/sys/fs/bpf/cloudwalker");
        }
    }
``` 
defer宏：程序退出时自动清理eBPF挂载目录/sys/fs/bpf/cloudwalker，确保资源释放。  

3.  Openssl检测 

```rust
let path = check_openssl().context("failed to get openssl").unwrap();
    info!(path = ?path.display(), "found openssl");
    let version = detect_openssl_version(&path)
        .context("failed to detect openssl version")
        .unwrap();
    info!(version, "detected openssl version");
``` 

路径检测：调用check_openssl查找系统OpenSSL路径。    

版本检测：通过detect_openssl_version获取版本号，并记录日志。        

4. 配置eBPF探针 

```rust
let opt = UprobeOpt {
        func_name: PROBE_OPENSSL_FN.map(ToOwned::to_owned).collect(),
        binary_path: vec![path.to_str().unwrap().to_string()],
    };
    let opt_map: HashMap<String, UprobeOpt> = [("openssl".to_string(), opt)].into();
    let version = version.strip_prefix("openssl ").unwrap();
``` 

Uprobe配置：设置要探测的OpenSSL函数名（如SSL_write/SSL_read）和二进制路径。 

版本处理：去除版本字符串前缀，得到纯版本号（如1.1.1k）。    

5. 动态加载eBPF程序 

```rust
let bpf = match version {
        o102 if o102.starts_with("1.0.2") => load_openssl102a_ebpf(opt_map).unwrap(),
        o110 if o110.starts_with("1.1.0") => load_openssl110a_ebpf(opt_map).unwrap(),
        o111x if o111x.starts_with("1.1.1") => { /* 处理子版本 */ },
        o30x if o30x.starts_with("3.0.") => load_openssl300_ebpf(opt_map).unwrap(),
        // ...其他版本处理
    };
```

版本适配：根据检测到的OpenSSL版本，加载对应的预编译eBPF程序，确保兼容性。   

6. 事件处理循环 

```rust
let mut r = bpf.receiver.resubscribe();
    let exit = tokio::signal::ctrl_c();
    pin!(exit);
    loop {
        select! {
        ev = r.recv() => { /* 处理事件 */ },
            _ = &mut exit => { /* 处理Ctrl-C退出 */ },
        }
    }
``` 

异步事件循环：使用tokio::select!同时监听eBPF事件和Ctrl-C信号。  

优雅退出：用户按下Ctrl-C时打印信息并返回成功状态码。    

7. TLS数据解析     

```rust
match ev.version {
        ciphers::TLS12_VERSION => {
            info!("{}\nCR  {}\nMK  {}", ev.version.name(), hex::encode(ev.client_random), hex::encode(ev.master_key));
        },
        ciphers::TLS13_VERSION => {
            info!("{} ...", ev.version.name()); // 输出更多密钥信息
        },
        _ => error!("unknown tls version"),
    }
``` 
TLS版本处理：   
TLS 1.2及以下：记录客户端随机数（Client Random）和主密钥（Master Key）。     
TLS 1.3：记录握手密钥、流量密钥等更复杂的参数。 
未知版本：记录错误日志。    

**关键点总结**     

eBPF动态探测：通过用户态与内核态协作，动态注入探针到OpenSSL关键函数（如SSL_read/SSL_write），捕获TLS握手数据。  

多版本支持：覆盖OpenSSL 1.0.2到3.4.x的多个版本，适配不同符号和内部结构。    

异步架构：利用Tokio运行时实现高效事件循环，处理高并发网络流量。 

安全清理：通过defer!确保eBPF资源释放，避免残留导致系统问题。    

密钥提取：核心功能是提取TLS连接中的关键安全参数，可用于解密流量或安全审计。 

## 性能事件监控

在这段代码中，“性能事件”（Performance Events）是指Linux内核提供的机制，用于监视和收集关于硬件和软件事件的数据。 

这些事件可以用来分析系统的性能瓶颈、调试问题以及进行性能优化。

### 性能事件的主要特点  

#### 硬件事件：

这些事件由处理器硬件记录，例如缓存未命中、分支预测失败、TLB未命中等。   

#### 软件事件：  

这些事件由操作系统内核记录，例如上下文切换、任务调度、系统调用等。  

### 性能事件的应用场景

#### 性能分析：

通过监视特定的性能事件，可以了解应用程序或系统在运行过程中的性能特征。  

#### 调试： 

可以帮助诊断性能问题，例如识别导致高延迟的操作。    

#### 优化： 

通过对性能事件的监控，可以找出需要优化的部分，提高系统的整体性能。  

在这段代码中，性能事件主要用于监视和收集系统运行时的性能数据。  

通过这些数据，开发者可以更好地理解系统的运行状态，从而进行性能优化和调试  

## 内存会计(Memory Accounting)  

### 什么是内存会计  

内存会计是指操作系统对内存使用情况进行记录和管理的过程。        

它涉及跟踪每个进程或用户使用的内存数量，并根据一定的策略进行资源分配和控制。        

内存会计的主要目的是确保系统资源的合理分配和利用，防止某个进程占用过多内存导致其他进程无法正常运行。

### 内存会计的实现方式  

内存限制：为每个进程或用户设置内存使用上限，超过这个上限则不允许继续分配内存。  

内存统计：记录每个进程的内存使用情况，包括已用内存、峰值内存使用量等。  

内存调度：根据内存使用情况动态调整进程的优先级，确保关键任务有足够的内存资源。  

### 示例    

假设有一个多任务的操作系统，其中每个进程都有自己的内存限制。        

操作系统会定期检查每个进程的内存使用情况，如果某个进程超过了其内存限制，则操作系统可能会采取以下措施之一：  

页面置换：将该进程的部分内存页移到交换分区，释放物理内存。  

杀死进程：如果页面置换后仍然没有足够的内存，操作系统可能会选择终止该进程。  

内存回收：回收其他进程的缓存和缓冲区，为当前进程腾出更多内存。  

通过这种方式，操作系统可以有效地管理和控制内存资源，确保系统的稳定性和性能。   

### 实际代码    

这段 Rust 代码用于检测当前 Linux 内核是否支持内核内存（kmem）会计功能， 

并根据检测结果调整进程的内存锁定限制（RLIMIT_MEMLOCK）。    

以下是对代码的详细解析及其作用的说明：  

1.  detect_kmem_accouting函数   

用于检测当前内核是否支持内核内存会计功能    

```rust
//  获取当前的memlock限制
let ret = unsafe {
    prlimit64(
        0 as pid_t,
        RLIMIT_MEMLOCK,
        null(),
        &mut old_limit as *mut rlimit64,
    )
};
if ret != 0 {
    return Err(Error::last_os_error());
}
//  使用 prlimit64 系统调用获取当前进程的 RLIMIT_MEMLOCK 值，并存储在 old_limit 中。
```

```rust
//  临时将 memlock 软限制设置为零：
let zero_limit: rlimit64 = rlimit64 {
    rlim_cur: 0,
    rlim_max: old_limit.rlim_max, // 不改变最大值，以便之后恢复
};
let ret = unsafe {
    prlimit64(
        0 as pid_t,
        RLIMIT_MEMLOCK,
        &zero_limit as *const rlimit64,
        &mut old_limit as *mut rlimit64,
    )
};
if ret != 0 {
    return Err(Error::last_os_error());
}
// 将 memlock 的软限制临时设置为零，以测试内核是否支持内存会计。
``` 

检测内核内存会计支持：

通过临时将 memlock 软限制设置为零，并尝试创建一个 BPF 映射来判断内核是否支持内存会计。如果创建成功，说明内核支持内存会计；否则，不支持。
调整进程的内存锁定限制：

如果内核支持内存会计，无需进一步操作。
如果内核不支持内存会计，函数会将进程的 memlock 软限制设置为无穷大，以允许进程锁定更多的内存。这通常是为了确保某些需要锁定内存的操作（如使用 BPF）能够成功执行。
日志记录和错误处理：

使用 tracing 库记录检测过程和操作结果，便于调试和监控。
通过详细的错误处理，确保在操作失败时能够提供有用的错误信息。
使用场景和注意事项
使用场景：

该代码通常用于需要使用 BPF 或其他需要锁定内存的内核功能的应用程序中。通过检测和调整 memlock 限制，确保这些功能能够正常运行。
注意事项：

修改 RLIMIT_MEMLOCK 可能会影响系统的安全性，特别是将其设置为无穷大时，可能导致系统资源耗尽。因此，只有在确实需要时才应进行此类操作，并确保在操作完成后恢复原始限制。    


