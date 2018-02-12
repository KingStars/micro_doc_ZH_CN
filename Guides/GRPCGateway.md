# GRPC网关
本指南有助于将grpc网关与go-micro服务结合使用。

[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)是[protoc](http://github.com/google/protobuf)的插件。它读取[gRPC](http://github.com/grpc/grpc-common)服务定义，并生成将RESTful JSON API转换为gRPC的反向代理服务器。

我们使用[go-grpc](https://github.com/micro/go-grpc)编写后端服务。Go-GRPC是围绕go-micro和用于客户端和服务器的grpc插件的简单包装。当调用[grpc.NewService](https://godoc.org/github.com/micro/go-grpc#NewService)时，它返回一个[micro.Service](https://godoc.org/github.com/micro/go-micro#Service)。

## code
在[examples/grpc](https://github.com/micro/examples/tree/master/grpc)找到示例代码。

## 前提
这些是一些先决条件

### 安装protobuf

```
mkdir tmp
cd tmp
git clone https://github.com/google/protobuf
cd protobuf
./autogen.sh
./configure
make
make check
sudo make install
```

### 安装插件

```
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
go get -u github.com/micro/protobuf/protoc-gen-go
```

## Greeter服务
在这个例子中，我们使用go-grpc创建了一个Greeter的微服务。 这项服务非常简单。

原型如下：

```
syntax = "proto3";

package go.micro.srv.greeter;

service Say {
	rpc Hello(Request) returns (Response) {}
}

message Request {
	string name = 1;
}

message Response {
	string msg = 1;
}
```

服务如下：

```
package main

import (
	"log"
	"time"

	hello "github.com/micro/examples/greeter/srv/proto/hello"
	"github.com/micro/go-grpc"
	"github.com/micro/go-micro"

	"golang.org/x/net/context"
)

type Say struct{}

func (s *Say) Hello(ctx context.Context, req *hello.Request, rsp *hello.Response) error {
	log.Print("Received Say.Hello request")
	rsp.Msg = "Hello " + req.Name
	return nil
}

func main() {
	service := grpc.NewService(
		micro.Name("go.micro.srv.greeter"),
		micro.RegisterTTL(time.Second*30),
		micro.RegisterInterval(time.Second*10),
	)

	// optionally setup command line usage
	service.Init()

	// Register Handlers
	hello.RegisterSayHandler(service.Server(), new(Say))

	// Run server
	if err := service.Run(); err != nil {
		log.Fatal(err)
	}
}
```

## GRPC网关
grpc网关使用与服务相同的协议，并增加一个http选项

```
syntax = "proto3";

package greeter;

import "google/api/annotations.proto";

service Say {
	rpc Hello(Request) returns (Response) {
		option (google.api.http) = {
			post: "/greeter/hello"
			body: "*"
		};
	}
}

message Request {
	string name = 1;
}

message Response {
	string msg = 1;
}
```

proto使用以下命令生成grpc stub和反向代理

```
protoc -I/usr/local/include -I. \
  -I$GOPATH/src \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --go_out=plugins=grpc:. \
  path/to/your_service.proto
```

```
protoc -I/usr/local/include -I. \
  -I$GOPATH/src \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --grpc-gateway_out=logtostderr=true:. \
  path/to/your_service.proto
```

我们使用下面的代码创建了greeter服务的示例api。将写入类似的代码来注册其他端点。请注意，网关需要greeter服务的端口地址。

```
package main

import (
	"flag"
	"net/http"

	"github.com/golang/glog"
	"github.com/grpc-ecosystem/grpc-gateway/runtime"
	"golang.org/x/net/context"
	"google.golang.org/grpc"

	hello "github.com/micro/examples/grpc/gateway/proto/hello"
)

var (
	// the go.micro.srv.greeter address
	endpoint = flag.String("endpoint", "localhost:9090", "go.micro.srv.greeter address")
)

func run() error {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	mux := runtime.NewServeMux()
	opts := []grpc.DialOption{grpc.WithInsecure()}

	err := hello.RegisterSayHandlerFromEndpoint(ctx, mux, *endpoint, opts)
	if err != nil {
		return err
	}

	return http.ListenAndServe(":8080", mux)
}

func main() {
	flag.Parse()

	defer glog.Flush()

	if err := run(); err != nil {
		glog.Fatal(err)
	}
}
```

## 运行示例
运行greeter服务。指定mdns，因为我们不需要发现。

```
go run examples/grpc/greeter/srv/main.go --registry=mdns --server_address=localhost:9090
```

运行网关。它将默认为端点localhost:9090的greeter服务。

```
go run examples/grpc/gateway/main.go
```

使用curl在（localhost:8080）网关上发起请求。

```
curl -d '{"name": "john"}' http://localhost:8080/greeter/hello
```

## 限制
grpc网关的例子需要提供服务地址，而我们自己的micro api使用服务发现，动态路由和负载均衡。这使得grpc网关的集成性稍差一些。

访问[github.com/micro/micro](https://github.com/micro/micro)了解更多信息
