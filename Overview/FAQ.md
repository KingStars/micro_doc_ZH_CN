# 常问问题
常见问题解答应该为最常见的问题提供快速解答。

## 什么是Micro？
Micro是一个专注于简化分布式系统开发的微服务生态系统。

- Micro是一个[框架](https://github.com/micro/go-micro)
- Micro是一个[工具包](https://github.com/micro/micro)
- Micro是一个[社区](http://slack.micro.mu/)
- Micro是一个[生态系统](https://micro.mu/explore/)

### 开源
Micro由开放源码库和工具组成，以帮助微服务开发。

- **go-micro** - 用于编写微服务的可插入Go RPC框架; 服务发现，客户端/服务器rpc，pub/sub等。
- **go-plugins** - go-micro的插件，包括etcd，kubernetes，nats，rabbitmq，grpc等
- **micro** - 一个包含传统入口点的微服务工具包; API网关，CLI，Slack Bot，Sidecar和Web UI。

其他各种库和服务可以在[github.com/micro](https://github.com/micro)找到。

### 社区
有一个有千名会员的松散社区。

在[slack.micro.mu](http://slack.micro.mu/)邀请你自己。

### 生态系统
Micro跨越单一组织。开源工具和服务正在由社区自己提供。

在[micro.mu/explore/](https://micro.mu/explore/)上探索生态系统。

## 我从哪里开始？
从[go-micro](https://github.com/micro/go-micro)开始。自述文件提供了一个微服务示例。

阅读[入门指南](https://micro.mu/docs/writing-a-go-service.html)或查看[示例](https://github.com/micro/examples)，了解更多信息。

使用[micro](https://github.com/micro/micro)工具包，通过cli，web ui，slack或api网关访问微服务。

## 谁在使用Micro？
在[用户](https://micro.mu/docs/users.html)页面查看使用Micro的公司列表，（但注意它可能已过时）。

还有很多人也在使用它，但尚未公开列出。如果您使用Micro，请随时添加您的公司。

## 我如何使用Micro？
这很简单。

1. 使用[go-micro](https://github.com/micro/go-micro)编写服务。
2. 通过[micro](https://github.com/micro/micro)工具包访问它们。
3. 完成。

检查完整的[greeter](https://github.com/micro/examples/tree/master/greeter)示例。

## 我可以替代Consul吗？
可以! 服务发现注册表与其他所有软件包一样，是完全可插入的。由于其特点和简单性，Consul被用作默认值。

### ETCD
举个例子。如果您想使用etcd，请导入插件并在二进制文件中设置命令行标志。

```
import (
        _ "github.com/micro/go-plugins/registry/etcd"
)
```

```
service --registry=etcd --registry_address=127.0.0.1:2379
```

### 零依赖
有一个内置的零依赖的Multicast DNS服务注册表配置。在启动时将 `--registry=mdns` 或 `MICRO_REGISTRY=mdns` 传递给您的应用程序即可。

## 我可以在哪里运行Micro？
Micro是运行时不感知的。你可以在任何你喜欢的地方运行它。裸机上AWS，谷歌云。在你最喜欢的容器编排系统，如Mesos或Kubernetes。

事实上，在Kubernetes上有Micro的演示配置。查看[https://github.com/micro/kubernetes](https://github.com/micro/kubernetes)。

## API，Web和SRV服务有什么区别？

![](https://micro.mu/docs/images/arch.png)

作为Micro工具包的一部分，我们尝试通过分离API，Web仪表盘和后端服务（SRV）的关注点，为可扩展体系结构定义一组设计模式。

### API服务
API服务由Micro Api提供，默认命名空间为go.micro.api。micro API符合API网关模式。

点击[此处](https://github.com/micro/micro/tree/master/api)了解详情

### Web服务
Web服务由Micro Web提供，默认名称空间为go.micro.web。我们相信web应用程序是微服务世界中的一等公民，因此可以将web仪表板作为微服务来构建。Micro网络是一个反向代理，它会根据服务解析的路径将HTTP请求转发到相应的Web应用程序。

点击[此处](https://github.com/micro/micro/tree/master/web)了解详情

### SRV服务
SRV服务基本上是标准的RPC服务，通常这是你写的服务。我们通常称它们为RPC或后端服务，因为它们主要应该是后端架构的一部分，并且永远不会面向公众。默认情况下，我们使用命名空间go.micro.srv，但是您应该使用您的域com.example.srv。

## 它性能如何？
性能不是Micro的当前焦点。尽管代码编写为最佳并避免了开销，但基准测试并没有花费太多时间。与net/http或其他web框架进行比较是没有意义的。Micro为包括服务发现，负载平衡，消息编码等的微服务提供了更高级别的要求。为了比较，您需要将所有这些功能添加进来。

如果你仍然关心性能。提取最大值的最简单方法是简单地通过运行以下标志：

```
--selector=cache # enables in memory caching of discovered nodes
--client_pool_size=10 # enables the client side connection pool
```

## Micro是否支持gRPC？
是的。在[micro/go-plugins](https://github.com/micro/go-plugins)中有传输，客户端和服务器的插件。

如果你想快速入门，只需使用[micro/go-grpc](https://github.com/micro/go-grpc)。

## Micro与Go-Kit
这个问题出现了很多。micro和go-kit有什么区别？

Go-kit将自己描述为微服务的标准库。像Go一样，go-kit为您提供可用于构建应用程序的单独包。如果您想完全控制您定义服务的方式，Go-kit非常棒。

Go-micro是微服务的可插入RPC框架。这是一个自发的框架，试图简化分布式系统的通信方面，以便您可以专注于业务逻辑本身。 Go-micro非常适合您快速启动和运行，同时拥有可插拔的基础架构而无需更改代码。

Micro是一个微服务工具包。这就像微型服务的瑞士军刀一样，微型服务以go-micro为基础，提供诸如http api gateway，web ui，cli，slack bot等传统入口点。Micro使用工具来指导逻辑上分离您的架构中的关注点，从而推动您为公共API创建一个微服务的API层，并为Web UI分别创建一个微服务的WEB层。

在想要完全控制的地方使用go-kit。你想要一个自用的框架使用go-micro。

## 我在哪里可以了解更多？
- **加入松散社区** - [slack.micro.mu](http://slack.micro.mu/)
- **阅读博客** - [micro.mu/blog](https://micro.mu/blog)
- **如果你想谈谈** - [contact@micro.mu](mailto:contact@micro.mu)
