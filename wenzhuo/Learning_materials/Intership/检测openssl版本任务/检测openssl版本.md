**实习任务内容: 基于ecapture开源项目,用rust重写golang版本的openssl版本检测.【技术栈:rust、elf文件格式】**   


**解决了什么问题：通过将版本字符串映射到具体的 BPF 文件，程序可以根据检测到的 OpenSSL 版本加载正确的 BPF 文件，从而确保能够正确地处理和拦截 OpenSSL 相关的网络流量【tls/ssl】。**     

