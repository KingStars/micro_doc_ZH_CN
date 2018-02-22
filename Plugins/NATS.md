# NATS
这篇文档着眼于nats，将nats消息系统集成到micro工具包中。它讨论包括围绕服务发现，微服务的同步和异步通信。

## 什么是NATS？
[NATS](http://nats.io/)是一个开源的原生云消息系统或更简单的消息总线。NATS由[Apcera](https://www.apcera.com/)的创始人Derek Collison创建。它起源于VMWare，开始基于ruby的一个系统。基于Go重写了很长时间，当前正在稳步进行，并且获得那些寻求高度扩展和高性能消息传递系统的人们所采用。

如果您想了解有关NATS的更多信息，请访问[nats.io](http://nats.io/)或加入[社区](http://nats.io/community/)。

## 为什么选择NATS？
为什么不NATS？过去曾与许多消息总线合作过，很快就清楚NATS是分开的。多年来，消息总线被誉为企业的救星，导致系统试图成为所有人的一切。这导致了很多虚假的承诺，显着的性能膨胀和高成本技术，产生了比解决问题更多的问题。

相比之下，NATS采取非常专注的方式，解决性能和可用性问题，同时保持令人难以置信的精益。它提到“永远在线并且可用”，并且使用“消防和忘记”消息模式。它的简单性，重点和轻量级特性使其成为微服务生态系统的主要候选者。我们相信它很快将成为服务间消息传递的主要候选人。

NATS提供了什么：
- 高性能和可扩展性
- 高可用性
- 极其轻巧
- 最多一次交付

什么NATS不提供：
- 持久化
- 事务处理
- 增强交付模式
- 企业级队列

这简要介绍了选择NATS的原因。那么它如何适应Micro？来！我们讨论一下。

### Micro on NATS
[Micro](https://github.com/micro/micro)是一个微服务工具包，采用可插拔的体系结构，允许将底层的依赖关系以最小的更改进行替换。[Go-Micro](https://github.com/micro/go-micro)框架的每个接口都为微服务提供了构建块；用于服务发现的[注册表](https://godoc.org/github.com/micro/go-micro/registry#Registry)，用于同步通信的[传输](https://godoc.org/github.com/micro/go-micro/transport#Transport)，用于异步[消息总线](https://godoc.org/github.com/micro/go-micro/broker#Broker)的代理等等。

为每个组件创建插件并实现接口一样简单。我们将花更多时间详细说明，如何在未来的博客文章中撰写插件。如果您想查看NATS或任何其他系统的插件，例如etcd discovery，kafka broker，rabbitmq transport，您可以在这里找到它们[github.com/micro/go-plugins](https://github.com/micro/go-plugins)。

基于NATS的Micro本质上是一组go-micro插件，可用于与NATS消息总线系统集成。通过为go-micro的各种接口提供插件，我们创建了许多可以选择的架构模式集成。

根据我们的经验，一种尺寸不适合所有情况，而Micro NATS的灵活性允许您定义适合您和您的团队模型。

下面我们将讨论传输，代理和注册中心的NATS插件的实现。

### 传输

![](nats-req-rsp.png)

传输是同步通信的Micro接口。它使用相当普通的Socket语义，与`Listen`，`Dial`和`Accept`类似。这些概念和模式对于使用tcp，http等的同步通信很好理解，但适应消息总线可能有些困难。与消息总线建立连接，而不是与服务本身建立连接。为了解决这个问题，我们使用与topics和channels的伪连接的概念。

这是它的工作原理。

服务使用`transport.Listen`来侦听消息。这将创建一个到NATS的连接。当`transport.Accept`被调用时，一个独特的topic被创建和订阅。这个独特的topic将被用作go-micro注册表中的服务地址。接收到的每条消息将被用作伪套接字/连接的基础。如果现有连接具有相同的回复地址，我们只需将该消息放入该连接的缓存中。

想要与此服务通信的客户端将使用`transport.Dial`创建与服务的连接。 这将连接到NATS，创建它自己独特的topic并订阅它。该topic用于服务的响应。每当客户端向服务发送消息时，它都会将回复地址设置为该topic。

当任何一方想要关闭连接时，他们只需调用`transport.Close`即可终止与NATS的连接。

![](nats-transport.png)

**使用传输插件**

导入传输插件

```
import _ "github.com/micro/go-plugins/transport/nats"
```

从传输标志开始

```
go run main.go --transport=nats --transport_address=127.0.0.1:4222
```

或者直接使用传输工具

```
transport := nats.NewTransport()
```

go-micro传输接口

```
type Transport interface {
    Dial(addr string, opts ...DialOption) (Client, error)
    Listen(addr string, opts ...ListenOption) (Listener, error)
    String() string
}
```

[Transport](https://github.com/micro/go-plugins/tree/master/transport/nats)

### Broker

![](nats-pub-sub.png)

代理是异步消息的转发micro接口。它提供了适用于大多数邮件代理的高级通用实现。就其本质而言，NATS是一个异步消息传输系统，它被用作消息代理。只有一个警告，NATS不持久化消息。虽然这对一些人来说可能并不理想，但我们仍然认为NATS可以也应该作为一个代理来使用go-micro。在不需要持久性的情况下，它允许高度可扩展的pub/sub子体系结构。

NATS提供了一个非常直接的发布和订阅机制，包括topic，channel等概念。在这里没有真正的花哨工作来使它工作。消息可以以异步火灾和遗忘的方式发布。使用相同channel名称的订阅者在NATS中形成一个队列组，然后它将允许消息自动均匀分布在订阅者中。

![](nats-broker.png)

**使用代理插件**

导入代理插件

```
import _ "github.com/micro/go-plugins/broker/nats"
```

从代理标志开始

```
go run main.go --broker=nats --broker_address=127.0.0.1:4222
```

或者直接使用代理

```
broker := nats.NewBroker()
```

go-micro 代理接口：

```
type Broker interface {
    Options() Options
    Address() string
    Connect() error
    Disconnect() error
    Init(...Option) error
    Publish(string, *Message, ...PublishOption) error
    Subscribe(string, Handler, ...SubscribeOption) (Subscriber, error)
    String() string
}
```
[Broker](https://github.com/micro/go-plugins/tree/master/broker/nats)

### 注册表

![](nats-service-discovery.png)

注册表是服务发现的Go-Micro界面。你可能会想。使用消息总线进行服务发现？这能工作吗？事实上它确实并且相当好。许多使用消息总线进行传输的人，会避免使用任何单独的发现机制。这是因为消息总线本身可以处理通过topic和channel的路由。定义为服务名称的topic可以用作路由key，在订阅该topic的服务的实例之间自动进行负载平衡。

Go-micro将服务发现和传输机制视为两个不同的问题。无论何时，客户端向另一个服务器发出请求（在封面下方），它会按名称在注册表中查找服务，选择节点的地址，然后通过传输器与其进行通信。

通常存储服务发现信息的最常用方式是通过
一个分布式的键值存储，如zookeeper，etcd或类似的东西。正如你可能已经意识到的那样，NATS不是一个分布式的键值存储，所以我们要做一些有点不同的事情......

**广播查询！**

广播查询与您想象的一样。 服务监听我们认为用于广播查询的特定主题。 任何想要获得服务发现信息的人都会首先创建一个它所订阅的回复主题，然后用他们的回复地址在广播主题上进行查询。

因为我们实际上并不知道有多少服务实例正在运行，或者有多少响应将被返回，所以我们设定了一个我们愿意等待响应的时间的上限。 这是发现分散聚集的粗糙机制，但由于NAT的可扩展和高性能特性，它实际上运行得非常好。 它也间接提供了一个非常简单的过滤服务的方法，并且响应时间更长。 将来，我们将着眼于改进底层实现。

总结它的工作原理：

1. 创建回复topic并订阅
2. 用广播地址发送广播topic的查询
3. 听取回复并在时间限制后取消订阅
4. 合并响应和返回结果

![](nats-registry.png)

**使用注册表插件**

导入注册表插件

```
import _ "github.com/micro/go-plugins/registry/nats"
```

从注册表标志开始

```
go run main.go --registry=nats --registry_address=127.0.0.1:4222
```

或者直接使用注册表

```
registry := nats.NewRegistry()
```

go-micro注册表接口：

```
type Registry interface {
    Register(*Service, ...RegisterOption) error
    Deregister(*Service) error
    GetService(string) ([]*Service, error)
    ListServices() ([]*Service, error)
    Watch() (Watcher, error)
    String() string
}
```

[Registry](https://github.com/micro/go-plugins/tree/master/registry/nats)

### 在NATS上伸缩Micro服务

在上面的例子中，我们只在本地主机上指定了一个NATS服务器，但我们推荐的实际使用方法是，设置一个NATS群集以获得高可用性和容错性。要了解有关NAT群集的更多信息，请查看[此处](http://nats.io/documentation/server/gnatsd-cluster/)的NATS文档。

Micro接受逗号分隔的地址列表作为上面提到的标志或可选地使用环境变量。如果您直接使用客户端库，它还允许一组可变主机作为注册表，传输和代理的初始化选项。

对于云原生应用的世界架构而言，我们过去的经验表明，每个AZ或每个地区的群集都是理想的。大多数云提供商在AZ之间具有相对较低（3-5ms）的延迟，这允许区域集群没有问题。在运行高可用性配置时，确保您的系统能够容忍AZ故障并且在更成熟的配置下，可以承受整个区域故障很重要。我们不建议跨地区集群。理想情况下，应该使用更高级别的工具来管理多群集和多区域系统。

Micro是一个令人难以置信的灵活的，运行时不感知的微服务系统。它可以在任何地方和配置下运行。它的世界是由服务注册机制引导。服务集群可以完全基于您提供访问权限的服务注册中心，AZ或区域池中进行本地化和命名空间。结合NATS集群，您可以构建高度可用的体系结构以满足您的需求。

![](region.png)

### 概要
[NATS](http://nats.io/)是一个可扩展的高性能消息中间件系统，我们认为它非常适合微服务生态系统。它与Micro配合具有非常好竞争力，我们已经证明它可以用作[注册表](https://godoc.org/github.com/micro/go-plugins/registry/nats)，[传输](https://godoc.org/github.com/micro/go-plugins/transport/nats)或[代理](https://godoc.org/github.com/micro/go-plugins/broker/nats)的插件。我们已经实施了所有三项，以突出NATS的灵活性。

Micro on NATS，是Micro强大的可插拔架构的一个典型例子。每个go-micro软件包都可以通过最小的更改来实现和替换。将来查看更多关于[X]的Micro示例。接下来最有可能是Kubernetes上的Micro。

希望这会激励你尝试使用NAT上的Micro，甚至为其他系统编写一些插件并回馈给社区。

在[github.com/micro/go-plugins](https://github.com/micro/go-plugins)找到NATS插件的来源。
