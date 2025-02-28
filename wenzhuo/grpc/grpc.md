# GRPC


## proto文件语法

```protobuf
//这是在说明我们使用的是proto3语法
syntax = "proto3";

// 这部分的内容是关于最后生成的go文件是处于哪个目录哪个包中，.代表在当前目录生成，service代表了生成的go文件的包名是service
option go_package = ".;service";

// 然后我们需要定义一个服务，在这个服务中需要有一个方法，这个方法可以接受客户端的参数，再返回服务端的响应。
// 其实很容易可以看出，我们定义了一个service,称为SayHello,这个服务有一个rpc方法，名为SayHello.
// 这个方法会发送一个HelloRequest,然后返回一个HelloResponse。
service SayHello {
    rpc SayHello(HelloRequest) returns (HelloResponse) {}
}

// message 关键字，其实你可以理解成Golang的结构体。
// 这里比较特别的是变量后面的“赋值”。注意：这里并不是赋值，而是在定义这个变量在这个message中的位置。
message HelloRequest {
    string requestName = 1;
    // int64 age = 2;
}

message HelloResponse {
    string reponseMsg = 1;
}
``` 

< message
message : protobuf 中定义一个消息类型