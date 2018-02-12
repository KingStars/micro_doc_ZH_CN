# 安装指南

## 依赖
我们需要服务发现，所以让我们启动Consul（默认），或者通过[go-plugins](https://github.com/micro/go-plugins)替换。

### Consul

```
brew install consul
consul agent -dev
```

或者

```
docker run consul
```


### Multicast DNS
我们可以使用Multicast DNS进行零依赖的服务发现

将`--registry=mdns`传递给任何命令，例如`micro --registry = mdns list services`

## Go Micro
Go Micro是Go开发微服务的RPC框架

### 安装

```
go get github.com/micro/go-micro
```

### Protobuf

如果您使用代码生成，您还需要使用protoc-gen-go

```
go get github.com/micro/protobuf/{proto,protoc-gen-go}
```

访问[github.com/micro/go-micro](https://github.com/micro/go-micro)了解更多。


## 工具包
Micro工具包提供了访问微服务的各种方法

### 安装

```
go get github.com/micro/micro
```

### Docker
可用预制docker images

```
docker pull microhq/micro
```

### 尝试CLI

运行greeter服务

```
go get github.com/micro/examples/greeter/srv && srv
```

服务清单

```
$ micro list services
consul
go.micro.srv.greeter
```

获取服务

```
$ micro get service go.micro.srv.greeter
service  go.micro.srv.greeter

version 1.0.0

Id	Address	Port	Metadata
go.micro.srv.greeter-34c55534-368b-11e6-b732-68a86d0d36b6	192.168.1.66	62525	server=rpc,registry=consul,transport=http,broker=http

Endpoint: Say.Hello
Metadata: stream=false

Request: {
	name string
}

Response: {
	msg string
}
```

查询服务

```
$ micro query go.micro.srv.greeter Say.Hello '{"name": "John"}'
{
	"msg": "Hello John"
}
```
