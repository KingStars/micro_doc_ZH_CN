# 写一个Go服务

这是go-micro入门指南。

如果您首先喜欢更高级别的工具包概览，请查看介绍性博客文章https://micro.mu/blog/2016/03/20/micro.html。

## 写一个服务
顶级[服务接口](https://godoc.org/github.com/micro/go-micro#Service)是构建服务的主要组件。它将Go Micro的所有底层软件包整合到一个简单的接口中。

```
type Service interface {
    Init(...Option)
    Options() Options
    Client() client.Client
    Server() server.Server
    Run() error
    String() string
}
```

### 1.初始化
像使用`micro.NewService`一样创建一个服务。

```
import "github.com/micro/go-micro"

service := micro.NewService() 
```

选项可以在创建过程中传入。

```
service := micro.NewService(
        micro.Name("greeter"),
        micro.Version("latest"),
)
```

所有可用的选项可以在[这里](https://godoc.org/github.com/micro/go-micro#Option)找到。

Go Micro还提供了使用`micro.Flags`设置命令行标志的方法。


```
import (
        "github.com/micro/cli"
        "github.com/micro/go-micro"
)

service := micro.NewService(
        micro.Flags(
                cli.StringFlag{
                        Name:  "environment",
                        Usage: "The environment",
                },
        )
)
```

解析标志使用`service.Init`。另外访问标志使用`micro.Action`选项。

```
service.Init(
        micro.Action(func(c *cli.Context) {
                env := c.StringFlag("environment")
                if len(env) > 0 {
                        fmt.Println("Environment set to", env)
                }
        }),
)
```

Go Micro提供了预定义的标志，如果调用`service.Init`，它将被设置和解析。看到[这里](https://godoc.org/github.com/micro/go-micro/cmd#pkg-variables)的所有标志

### 2.定义API

我们使用protobuf文件来定义服务API接口。这是一种非常方便的方式来严格定义API并为服务器和客户端提供具体的类型。

这是一个示例定义。

greeter.proto

```
syntax = "proto3";

service Greeter {
	rpc Hello(HelloRequest) returns (HelloResponse) {}
}

message HelloRequest {
	string name = 1;
}

message HelloResponse {
	string greeting = 2;
}
```

在这里，我们定义了一个名为Greeter的服务处理程序，其中的方法Hello使用参数`HelloRequest`类型并返回`HelloResponse`。

### 3.生成API接口

我们使用protoc和protoc-gen-go为这个定义生成具体的go实现。

Go-micro使用代码生成来提供客户端桩方法来减少代码编写，就像gRPC一样。这是通过一个protobuf插件完成的，它需要一个[golang/protobuf](https://github.com/golang/protobuf)分支，可以在这里找到[github.com/micro/protobuf](https://micro.mu/docs/github.com/micro/protobuf)。


```
go get github.com/micro/protobuf/{proto,protoc-gen-go}
protoc --go_out=plugins=micro:. greeter.proto
```

生成的类型现在可以在请求时在服务器或客户端的处理程序中导入和使用。

这是生成代码的一部分。

```
type HelloRequest struct {
	Name string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
}

type HelloResponse struct {
	Greeting string `protobuf:"bytes,2,opt,name=greeting" json:"greeting,omitempty"`
}

// Client API for Greeter service

type GreeterClient interface {
	Hello(ctx context.Context, in *HelloRequest, opts ...client.CallOption) (*HelloResponse, error)
}

type greeterClient struct {
	c           client.Client
	serviceName string
}

func NewGreeterClient(serviceName string, c client.Client) GreeterClient {
	if c == nil {
		c = client.NewClient()
	}
	if len(serviceName) == 0 {
		serviceName = "greeter"
	}
	return &greeterClient{
		c:           c,
		serviceName: serviceName,
	}
}

func (c *greeterClient) Hello(ctx context.Context, in *HelloRequest, opts ...client.CallOption) (*HelloResponse, error) {
	req := c.c.NewRequest(c.serviceName, "Greeter.Hello", in)
	out := new(HelloResponse)
	err := c.c.Call(ctx, req, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

// Server API for Greeter service

type GreeterHandler interface {
	Hello(context.Context, *HelloRequest, *HelloResponse) error
}

func RegisterGreeterHandler(s server.Server, hdlr GreeterHandler) {
	s.Handle(s.NewHandler(&Greeter{hdlr}))
}
```

### 4.实现处理程序

服务器需要注册处理程序来处理请求。处理程序是具有符合签名
```
func(ctx context.Context, req interface{}, rsp interface{}) error
```
错误的公共方法的公共类型。正如你在上面看到的，Greeter接口的处理器签名看起来像这样。

```
type GreeterHandler interface {
        Hello(context.Context, *HelloRequest, *HelloResponse) error
}
```

以下是Greeter处理程序的实现。

```
import proto "github.com/micro/examples/service/proto"

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
	rsp.Greeting = "Hello " + req.Name
	return nil
}
```

该处理程序在您的服务中注册很像一个http.Handler。

```
service := micro.NewService(
	micro.Name("greeter"),
)

proto.RegisterGreeterHandler(service.Server(), new(Greeter))
```

您也可以创建一个双向流媒体处理程序。

### 5. 运行服务

该服务可以通过调用`server.Run`来运行。这导致服务绑定到配置中的地址（默认为第一个RFC1918接口和随机端口）并侦听请求。

这将另外注册服务注册表启动和注销时发出一个kill信号。

```
if err := service.Run(); err != nil {
	log.Fatal(err)
}
```

### 6. 完整的服务

greeter.go

```
package main

import (
        "log"

        "github.com/micro/go-micro"
        proto "github.com/micro/examples/service/proto"

        "golang.org/x/net/context"
)

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
        rsp.Greeting = "Hello " + req.Name
        return nil
}

func main() {
        service := micro.NewService(
                micro.Name("greeter"),
                micro.Version("latest"),
        )

        service.Init()

        proto.RegisterGreeterHandler(service.Server(), new(Greeter))

        if err := service.Run(); err != nil {
                log.Fatal(err)
        }
}
```

注意：服务发现机制将需要运行，以便服务可以注册被客户和其他服务发现。快速入门就在[这里](https://github.com/micro/go-micro#getting-started)。

## 写一个客户端

[客户端](https://godoc.org/github.com/micro/go-micro/client)软件包用于查询服务。在创建服务时，将包含一个与服务器使用的初始化包相匹配的客户端。

查询以上服务如下所示。

```
// create the greeter client using the service name and client
greeter := proto.NewGreeterClient("greeter", service.Client())

// request the Hello method on the Greeter handler
rsp, err := greeter.Hello(context.TODO(), &proto.HelloRequest{
	Name: "John",
})
if err != nil {
	fmt.Println(err)
	return
}

fmt.Println(rsp.Greeter)
```

`proto.NewGreeterClient`获取服务名称和用于发出请求的客户端。

完整的例子可以在[go-micro/examples/service](https://github.com/micro/examples/tree/master/service)中找到。