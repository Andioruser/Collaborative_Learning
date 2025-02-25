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
