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
