# GRPC

## 什么是GRPC?

在 gRPC 里客户端应用可以像调用本地对象一样直接调用另一台不同的机器上服务端应用的方法，  

使得您能够更容易地创建分布式应用和服务。    

与许多 RPC 系统类似，gRPC 也是基于以下理念：    

定义一个服务，指定其能够被远程调用的方法（包含参数和返回类型）。    

在服务端实现这个接口，并运行一个 gRPC 服务器来处理客户端调用。  

在客户端拥有一个存根能够像服务端一样的方法。    

![alt text](image.png)  

## 使用 protocol buffers    

gRPC 默认使用 protocol buffers，    

这是 Google 开源的一套成熟的结构数据序列化机制（当然也可以使用其他数据格式如 JSON）。   

正如你将在下方例子里所看到的，你用 proto files 创建 gRPC 服务， 

用 protocol buffers 消息类型来定义方法参数和返回类型。  

## proto文件语法    

项目实际代码    

```proto
// 使用proto3版本
syntax = "proto3";

// proto文件包名
package chaitin.cloudwalker.proto.vigilcore;

option go_package = "/proto";  
// 生成的go/cpp/rust文件存放的路径，grpc文件存放的路径

// 引入外部的包
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// message 关键字类似于struct 结构体
message YaraClientConfig {
    string socket_path = 1;
    string yara_timeout = 2;
    string cwengine_timeout = 3;
    string disable_callgraph = 4;
}

message FileInputInfo {
    string file_path = 1;
    bool avira_engine = 2;
    bool yara_engine = 3;
}

message FileScanResult {
    // oneof类似联合体,选择其中之一
    oneof scan_results {
        // 类似空值 NULL
        google.protobuf.Empty no_scan_results = 1; // 表示没有扫描结果
        AviraScanResultList avira_scan_results = 2;
        YaraScanResultList yara_scan_results = 3;
    }
}

message AviraScanResult {
    string name = 1;
    string malware_name = 2;
    string malware_info = 3;
    string malware_type = 4;
    bool removable = 5;
}

message AviraScanResultList {
    enum ScanStatus {
        Clean = 0;
        Infected = 1;
        Suspicious = 2;
        Error = 3;
        Finished = 4;
    }
    ScanStatus overall_status = 1;
    // 数组 []list 列表
    repeated AviraScanResult results = 2;
}

// message YaraVersion {
//     string engine_name = 1;
//     string version = 2;
// }

// message YaraMeta {
//     string engine_name = 1;
// }

// message YaraRuleSetResult {
//     bool success = 1;
// }

message YaraScanResult {
    uint32 risk_level = 1;
    string type = 2;
    string sub_type = 3;
    string name = 4;
    string file = 5;
    string reason = 6;
    string reason_en = 7;
}

message YaraScanResultList {
	google.protobuf.Timestamp time_cost = 1; 
	repeated YaraScanResult results = 2;
}

// service [关键字] 服务名
service ScanFile {
    // rpc [类似于 golang的func关键字] 函数名 (入参) returns (返回值) {} 
    rpc ScanFile(FileInputInfo) returns (FileScanResult) {}
}

```