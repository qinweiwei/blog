---
title: gRPC框架简介
date: 2018-03-28
updated: 2018-03-28
tags:
    - gRPC
    - protocol buffer
---

# gRPC框架简介
gRPC是google开源的一款高性能、开源和通用的RPC框架，面向移动和http/2设计，基于[Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview)序列化协议开发，且支持众多语言。
以下内容基本都是基于官网和其他文档整理，在这里只是记录便于后续查找。
<!-- more -->
## gRPC特性
- 强大的IDL特性: gRPC使用ProtoBuf来定义服务,ProtoBuf的相关内容见下边
- 支持多语言：支持多种语言，包括C，C++，Java，ruby，Python，Go等，并能自动生成客户端和服务端的功能库，支持不用语言构建的客户端和服务端通信
- 基于HTTP/2标准设计：gRPC基于HTTP/2标准设计，所以且支持HTTP/2的诸多特性，比如双向流，头压缩，多路复用等。

## Potocol Buffers
目前gRPC都是基于Protocol Buffers实现RPC框架，所以PotoBuf是我们必须要了解的，ProtoBuf是Google出品的一种轻量&高效的结构化数据的存储格式，类似json，XML，但性能却比Json，XML强太多，其通过将 结构化的数据 进行 串行化（序列化），从而实现数据存储/RPC数据交换的功能
### ProtoBuf特性
![ProtoBuf.png](https://i.loli.net/2020/03/17/NAgkHQunbJKGlLa.png)
### 如何使用ProtoBuf
- 定义`.proto`文件，在`.proto`文件中明确数据结构和rpc service
- 使用protobuf 编译器protoc根据`.proto`文件编译，生产相应语言的的代码
- 把相关代码保存到自己的工程中使用相关接口
### `.proto`文件的语法
发现一篇文章对`.proto`文件语法介绍的非常详细，在这里直接转载该文章[ProtoBuf语法](https://blog.csdn.net/carson_ho/article/details/70267574)或者直接查看[官网](https://developers.google.com/protocol-buffers/docs/proto3)
### 基于Go的ProtoBuf使用
1. 定义message格式在`.proto`文件中

```
syntax = "proto3";
package tutorial;

message Person {
  string name = 1;
  int32 id = 2;  // Unique ID number for this person.
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;
}

// Our address book file is just one of these.
message AddressBook {
  repeated Person people = 1;
}
```

2. 编译`.proto`文件
- 安装protoc，从[protobuf](https://github.com/google/protobuf/releases)下载protoc文件，解压后将protoc文件放入PATH路径
- 安装protobuf库文件：go get -u github.com/golang/protobuf/proto
- 安装插件：go get -u github.com/golang/protobuf/protoc-gen-go；export PATH=$PATH:$GOPATH/bin
- protoc -I=$SRC_DIR --go_out=$DST_DIR $SRC_DIR/addressbook.proto

3. 序列化和反序列化message
使用proto库: proto.Marshal，proto.Unmarshal

```
book := &pb.AddressBook{}
// ...

// Write the new address book back to disk.
out, err := proto.Marshal(book)
if err != nil {
        log.Fatalln("Failed to encode address book:", err)
}
if err := ioutil.WriteFile(fname, out, 0644); err != nil {
        log.Fatalln("Failed to write address book:", err)
}
```

```
// Read the existing address book.
in, err := ioutil.ReadFile(fname)
if err != nil {
        log.Fatalln("Error reading file:", err)
}
book := &pb.AddressBook{}
if err := proto.Unmarshal(in, book); err != nil {
        log.Fatalln("Failed to parse address book:", err)
}
```

## gRPC概念
- 服务定义：gRPC定义一个服务，指定其可以被远程调用的方法及其参数和返回类型。gPRC默认使用ProtoBufa作为接口定义语言，来描述服务接口和有效载荷消息结构。如果有需要的话，可以使用其他替代方案

```
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  required string greeting = 1;
}

message HelloResponse {
  required string reply = 1;
}
```

- gRPC支持的四类服务方法
    + 单项 RPC，即客户端发送一个请求给服务端，从服务端获取一个应答，就像一次普通的函数调用。
    + 服务端流式 RPC，即客户端发送一个请求给服务端，可获取一个数据流用来读取一系列消息。客户端从返回的数据流里一直读取直到没有更多消息为止。
    + 客户端流式 RPC，即客户端用提供的一个数据流写入并发送一系列消息给服务端。一旦客户端完成消息写入，就等待服务端读取这些消息并返回应答。
    + 双向流式 RPC，即两边都可以分别通过一个读写数据流来发送一系列消息。这两个数据流操作是相互独立的，所以客户端和服务端能按其希望的任意顺序读写，例如：服务端可以在写应答前等待所有的客户端消息，或者它可以先读一个消息再写一个消息，或者是读写相结合的其他方式。每个数据流里消息的顺序会被保持。
- 使用 API 接口: gRPC 提供 protocol buffer 编译插件，能够从一个服务定义的 .proto 文件生成客户端和服务端代码。通常 gRPC 用户可以在服务端实现这些API，并从客户端调用它们。
    + 在服务侧，服务端实现服务接口，运行一个 gRPC 服务器来处理客户端调用。gRPC 底层架构会解码传入的请求，执行服务方法，编码服务应答。
    + 在客户侧，客户端有一个存根实现了服务端同样的方法。客户端可以在本地存根调用这些方法，用合适的 protocol buffer 消息类型封装这些参数— gRPC 来负责发送请求给服务端并返回服务端 protocol buffer 响应。
## gRPC-go教程
该部分直接来自[官网](https://grpc.io/docs/tutorials/basic/go.html)或者[中文翻译](https://doc.oschina.net/grpc?t=60133)

