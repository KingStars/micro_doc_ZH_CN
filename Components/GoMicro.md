# Go Micro
Go Micro是一个插件式的RPC框架。它用于分布式系统开发。

![](go-micro.png)

## 特性
Go Micro抽象出分布式系统的细节。以下是主要功能。

- 服务发现 - 通过服务发现自动注册和名称解析
- 负载平衡 - 基于发现构建的服务的智能客户端负载平衡
- 同步通信 - 基于RPC的通信，支持双向流
- 异步通信 - 为事件驱动架构内置的Pub/Sub接口
- 消息编码 - 基于带有protobuf和json的内容类型的动态编码
- 服务接口 - 所有功能都打包在一个简单的高级界面中，用于开发微服务

Go Micro支持服务和功能编程模型。请继续阅读以了解更多信息。

## 插件
Go-micro使用Go界面进行抽象。由于这个原因，底层实现可以被换出。

我们提供了开箱即用的默认设置。

- Consul或mDNS服务发现
- 随机散列客户端负载平衡
- JSON-RPC 1.0和PROTO-RPC消息编码
- HTTP通信机制

## 示例
在[示例/函数](https://github.com/micro/examples/tree/master/function)以及[示例/服务](https://github.com/micro/examples/tree/master/service)可以找到示例服务。

[示例](https://github.com/micro/examples)目录包含使用诸如中间件/封装器，选择器过滤器，pub/sub，grpc，插件等的示例。对于完整的greeter示例，请查看[示例/greeter](https://github.com/micro/examples/tree/master/greeter)程序。其他示例可以在整个GitHub存储库中找到。

观看[Golang UK Conf 2016](https://www.youtube.com/watch?v=xspaDovwk34)视频以获取高级概述。

## 软件包
Go micro由多个软件包组成。

- 传输同步消息
- 代理异步消息
- 用于消息编码的编解码器
- 服务发现注册表
- 选择器进行负载平衡
- 客户端提出请求
- 服务器来处理请求

更多细节如下

### 注册机制
注册机制提供了一个服务发现机制来将名称解析为地址。它可以由consul，etcd，zookeeper，dns，gossip等提供支持。服务应该在启动时使用注册机制进行注册，并在关闭时取消注册。服务可以选择提供一个到期的TTL并在一段时间内重新注册以确保活跃，并且如果服务死亡，服务将被清除。

### 选择器
选择器是建立在注册表上的负载平衡抽象。它允许使用过滤器功能对“服务”进行“过滤”，并使用诸如random，roundrobin，leastconn等算法选择“选择”服务。客户端在请求时利用选择器。客户端将使用选择器而不是注册表，因为它提供了内置的负载平衡机制。

### 传输
传输是用于服务之间的同步请求/响应通信的接口。它类似于golang网络包，但提供更高层次的抽象，允许我们切换通信机制，例如http，rabbitmq，websockets，NATS。该传输也支持双向流式传输。这对客户端推送到服务器提供强大的能力。

### 代理
代理为消息代理提供异步发布/订阅通信的接口。这是事件驱动架构和微服务的基本要求之一。默认情况下，我们使用收件箱样式指向HTTP系统，来最小化基本所需的依赖的数量。但是，在go-plugins中有许多消息代理实现可用，例如RabbitMQ，NATS，NSQ，Google Cloud Pub Sub。

### 编解码器
编解码器用于编码和解码消息，然后再通过线路传输消息。这可能是json，protobuf，bson，msgpack等。这与其他大多数编解码器不同之处在于我们实际上也支持RPC格式。所以我们有JSON-RPC，PROTO-RPC，BSON-RPC等。它将客户端/服务器的编码分离出来，并提供了一个强大的方法来集成其他系统，如gRPC，Vanadium等。

### 服务器
服务器是编写服务的构建模块。在这里，您可以命名您的服务，注册请求处理程序，添加middeware等。该服务构建在上述软件包上，为服务请求提供统一接口。内置一个RPC系统的服务器。将来可能会有其他的实现。该服务器还允许您定义多个编解码器以提供不同的编码消息。

### 客户端
客户端提供一个接口来向服务发出请求。就像服务器一样，它建立在其他软件包上，以提供一个统一的界面，用于使用注册机制，根据名称查找服务，使用选择器进行负载平衡，使用代理进行传输和异步消息传输的同步请求。

上述组件被组合在微服务的顶层。

## 内部实现
以下是关于核心部件的内部工作的原理。

### service.Run()
Go-micro服务通过调用service.Run()来启动，

1. 在启动功能之前执行

    ```
     for _, fn := range s.opts.BeforeStart {
             if err := fn(); err != nil {
                     return err
             }
     }
    ```
2. 启动服务

    ```
    if err := s.opts.Server.Start(); err != nil {
             return err
    }
    ```
3. 注册和服务发现

    ```
     if err := s.opts.Server.Register(); err != nil {
             return err
     }
    ```
4. 在启动功能后执行

    ```
    for _, fn := range s.opts.AfterStart {
            if err := fn(); err != nil {
                    return err
            }
    }
    ```

### server.Start()
server.Start由service.Run调用

1. 调用transport.Listen监听连接

    ```
     ts, err := config.Transport.Listen(config.Address)
     if err != nil {
             return err
     }
    ```

2. 调用transport.Accept开始接收链路

    ```
     go ts.Accept(s.accept)
    ```

3. 调用broker.Connect开始处理链路消息

    ```
     config.Broker.Connect()
    ```
    
4. 等待退出信号，关闭运输并断开代理

    ```

     go func() {
             // wait for exit
             ch := <-s.exit
    
             // wait for requests to finish
             if wait(s.opts.Context) {
                     s.wg.Wait()
             }
    
             // close transport listener
             ch <- ts.Close()
    
             // disconnect the broker
             config.Broker.Disconnect()
     }()

    ```
    
## 编写服务
查看入门[文档](../Guides/WritingaGoService.md)
