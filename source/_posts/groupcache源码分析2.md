---
title: groupcache源码分析2:groupcachepb
date: 2020-03-10 22:23:33
tags: groupcache
---

groupcachepb打开后我们可以看到.proto文件和.pb.go文件，这玩意儿我们很熟悉，protobuf协议嘛，我们只需要关注.proto即可，.pb.go是基于他生成，关于protobuf的具体内容不详述，大家知道他是一个用来进行通信的协议即可。

<!-- more -->

1. 协议版本和包名。
``` go
syntax = "proto2";

package groupcachepb;
```
2. 请求消息结构，两个字符串，分别为group和key，必须字段。
``` go
message GetRequest {
  required string group = 1;
  required string key = 2;
}
```
3. 回复消息结构，[]byte和double类型，可选字段。
``` go
message GetResponse {
  optional bytes value = 1;
  optional double minute_qps = 2;
}
```
4. rpc服务定义，但是我貌似没有在代码中看到使用rpc的地方。
``` go
service GroupCache {
  rpc Get(GetRequest) returns (GetResponse) {
  };
}
```
