# 编写一个Go函数
这是开始使用go-micro函数的指南。函数是一次执行服务。

如果您首先喜欢更高级别的工具包概述，请查看介绍的博客文章 https://micro.mu/blog/2016/03/20/micro.html

## 编写一个函数
顶层[函数接口](https://godoc.org/github.com/micro/go-micro#Function)是go-micro中函数编程模型的主要组件。它封装了Service接口，同时提供一次执行。

```
// Function is a one time executing Service
type Function interface {
	// Inherits Service interface
	Service
	// Done signals to complete execution
	Done() error
	// Handle registers an RPC handler
	Handle(v interface{}) error
	// Subscribe registers a subscriber
	Subscribe(topic string, v interface{}) error
}
```

### 1.初始化
一个函数就像使用`micro.NewFunction`一样创建。

```
import "github.com/micro/go-micro"

function := micro.NewFunction() 
```

选项可以在创建过程中传入。

```
function := micro.NewFunction(
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

function := micro.NewFunction(
        micro.Flags(
                cli.StringFlag{
                        Name:  "environment",
                        Usage: "The environment",
                },
        )
)
```

解析标志使用`function.Init`。另外访问标志使用`micro.Action`选项。

```
function.Init(
        micro.Action(func(c *cli.Context) {
                env := c.StringFlag("environment")
                if len(env) > 0 {
                        fmt.Println("Environment set to", env)
                }
        }),
)
```

Go Micro提供了预定义的标志，如果调用了`function.Init`，它将被设置和解析。看到[这里](https://godoc.org/github.com/micro/go-micro/cmd#pkg-variables)的所有标志。

### 2.定义API
我们使用protobuf文件来定义API接口。这是严格定义API并提供服务端和客户端的具体类型的一种非常方便的方式。

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

在这里我们定义了一个名为Greeter的函数处理程序，其中的方法Hello使用参数`HelloRequest`类型并返回`HelloResponse`。

### 3.生成API接口

我们使用protoc和protoc-gen-go为这个定义生成具体的实现。

Go-micro使用代码生成来提供客户端桩方法来减少代码编写，就像gRPC一样。这是通过一个protobuf插件完成的，它需要一个[golang/protobuf](https://github.com/golang/protobuf)分支，可以在这里找到[github.com/micro/protobuf](https://micro.mu/docs/github.com/micro/protobuf)。


```
go get github.com/micro/protobuf/{proto,protoc-gen-go}
protoc --go_out=plugins=micro:. greeter.proto
```

生成的类型现在可以在请求时在服务端或客户端的处理程序中导入和使用。

这是生成的代码的一部分。

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
服务器要求注册处理程序来处理请求。处理程序是一种公共方法，符合签名
```
func(ctx context.Context, req interface{}, rsp interface{}) error
```

正如你上面看到的，Greeter接口的处理器签名看起来像这样。

```
type GreeterHandler interface {
        Hello(context.Context, *HelloRequest, *HelloResponse) error
}
```

这是一个Greeter处理程序的实现。

```
import proto "github.com/micro/examples/service/proto"

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
	rsp.Greeting = "Hello " + req.Name
	return nil
}
```

处理程序的注册很像一个`http.Handler`。

```
function := micro.NewFunction(
	micro.Name("greeter"),
)

proto.RegisterGreeterHandler(service.Server(), new(Greeter))
```

或者，函数接口提供了一个更简单的注册模式。

```
function := micro.NewFunction(
        micro.Name("greeter"),
)

function.Handle(new(Greeter))
```

您也可以使用Subscribe方法注册一个异步订阅方法。

### 5.运行功能
该函数可以通过调用function.Run来运行。这将导致它绑定到配置中的地址（默认是第一个RFC1918接口和随机端口）并监听请求。

这将另外注册功能与注册表启动和注销时发出一个kill信号。

```
if err := function.Run(); err != nil {
	log.Fatal(err)
}
```

一旦发出请求，函数将退出。您可以使用[micro run](https://micro.mu/docs/run.html)管理功能的生命周期。一个完整的例子可以在[examples/function](https://github.com/micro/examples/tree/master/function)中找到。

### 6.完整的功能

greeter.go

```
package main

import (
        "log"

        "github.com/micro/go-micro"
        proto "github.com/micro/examples/function/proto"

        "golang.org/x/net/context"
)

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
        rsp.Greeting = "Hello " + req.Name
        return nil
}

func main() {
        function := micro.NewFunction(
                micro.Name("greeter"),
                micro.Version("latest"),
        )

        function.Init()

	function.Handle(new(Greeter))
	
        if err := function.Run(); err != nil {
                log.Fatal(err)
        }
}
```

注意：服务发现机制将需要运行，以便函数可以注册以供希望查询的人发现。快速入门就在[这里](https://github.com/micro/go-micro#getting-started)。

## 写一个客户端
[客户端](https://godoc.org/github.com/micro/go-micro/client)软件包用于查询功能和服务。当您创建一个函数时，将包含一个与服务器使用的初始化包相匹配的客户端。

查询上述功能就像下面这样简单。

```
// create the greeter client using the service name and client
greeter := proto.NewGreeterClient("greeter", function.Client())

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

`proto.NewGreeterClient`接受函数名称和用于发出请求的客户端。

完整的例子可以在[go-micro/examples/function](https://github.com/micro/examples/tree/master/function)中找到。
