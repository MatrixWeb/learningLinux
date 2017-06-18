### 一 什么是phxrpc

> PhxRPC是微信后台团队推出的一个非常简洁小巧的RPC框架。剧微信后台公众号下面的评论，这是微信内部框架的精简版。所以说明白这个phxrpc调用原理，基本就明白了使用规则。

### 二 准备好材料

> 1. centos 64 位的环境，我环境是：3.10.0-327.36.3.el7.x86_64
> 2. 基本工具：yum git install autoconf automake libtool curl make g++ unzip

### 三 动手下载源码
>1.  从git下载源码：git clone --recursive https://github.com/tencent-wechat/phxrpc.git
>2. 创建 protobuf的源代码目录：mkdir -p third_party
>3. 下载protobubf的源码，建议是最新的，支持proto3的语法：git clone https://github.com/google/protobuf.git
>4. 按手册编译phxrpc: https://github.com/tencent-wechat/phxrpc/wiki/%E4%B8%AD%E6%96%87%E8%AF%A6%E7%BB%86%E7%BC%96%E8%AF%91%E6%89%8B%E5%86%8C
>5. 启动server：./search_main -c ./search_server.conf -d （在phxrpc/sample目录下）
>6. 用工具测试下：./search_tool_main -c search_client.conf -f PHXEcho -s "Hello world"
>7. 结果：
```c++
PHXEcho return 0
resp: {
value: "Hello world"
}
```
