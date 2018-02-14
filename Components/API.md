# Micro API
micro api是微服务的API网关。使用API[网关模式](http://microservices.io/patterns/apigateway.html)为您的服务提供一个入口点。 micro api提供HTTP并动态路由到适当的后端服务。

![](https://micro.mu/docs/images/api.png)

## 如何工作的
micro api构建在go-micro上，利用它进行服务发现，负载平衡，编码和基于RPC的通信。对API的请求通过HTTP提供，并通过RPC进行内部路由。

由于micro api在内部使用go-micro，因此它也支持插件，因此可以随时切换为kubernetes api的consul服务发现或gRPC的RPC。

## API
micro api提供了以下HTTP API

```
- /[service]/[method]	# HTTP paths are dynamically mapped to services
- /rpc			# Explicitly call a backend service by name and method
```

见下面的例子

## Handler

Handler是管理请求路由的HTTP处理程序。

默认Handler使用注册表中的端口元数据来确定服务路由。如果未找到所匹配的路由，它将回退到API处理程序。您可以使用[go-api](https://github.com/micro/go-api)配置注册路由。

该API有三种可配置的请求Handler。

1. API Handler：`/[service]/[method]`
    - 请求/响应：`api.Request/api.Response`
    - 该路径用于解析服务和方法
    - 请求通过API服务处理，API服务采用请求`api.Request`和响应`api.Response`类型
    - 请求/响应的定义可以在[go-api/proto](https://github.com/micro/go-api/blob/master/proto/api.proto)中找到
    - 请求/响应主体的内容类型可以是任何东西
    - 路由不可用的默认回退处理程序
    - 通过`--handler=api`设置

2. RPC Handler：`/[service]/[method]`
    - 请求/响应：`json/protobuf`
    - 使用go-micro客户端将请求主体转发为RPC请求的默认处理程序的替代方案
    - 允许使用具体的Go类型定义API处理程序。
    - 在不需要完全控制标题或请求/响应的情况下很有用
    - 可以用来运行单层后端服务，而不是其他API服务
    - 支持的内容类型`application/json`和`application/protobuf`
    - 通过`--handler=rpc`设置

3. 反向代理：`/[service]`
    - 请求/响应：http
    - 该请求经过反向代理到服务的路径的第一个处理
    - 这允许REST在API后面实现
    - 通过`--handler=proxy`设置

4. Event Handler：`/[topic]/[event]`
    - 异步处理程序向消息代理发布请求作为事件
    - 请求被格式化为[go-api/proto.Event](https://github.com/micro/go-api/blob/master/proto/api.proto#L28L39)
    - 通过`--handler=event`进行设置

或者，使用`/rpc`端口直接与任何服务通话 - 期望参数：`service`，`method`，`request`，可选接受`address`，以指定特定主机。

```
curl -d 'service=go.micro.srv.greeter' \
	-d 'method=Say.Hello' \
	-d 'request={"name": "Bob"}' \
	http://localhost:8080/rpc
```

在[github.com/micro/examples/api](https://github.com/micro/examples/tree/master/api)中查找工作示例。

### API Handler 请求/响应原型
API Handler是一个默认处理原型，服务也是基于该原型使用特定的请求和响应处理，可在[go-api/proto](https://github.com/micro/go-api/blob/master/proto/api.proto)中获得。这允许micro api将HTTP请求解析为RPC并返回到HTTP。


## 入门

### 安装

```
go get github.com/micro/micro
```

### 运行

```
micro api
```

### 通过ACME加密
通过使用letsencrypt的ACME，提供默认安全服务

```
micro --enable_acme api
```

可以指定一个主机白名单

```
micro --enable_acme --acme_hosts=example.com,api.example.com api
```

### 提供安全的TLS
该API支持使用TLS证书安全地提供服务

```
micro --enable_tls --tls_cert_file=/path/to/cert --tls_key_file=/path/to/key api
```

### 设置命名空间
该API默认为服务名称空间`go.micro.api`。命名空间和请求路径的组合用于解析发送查询的API服务和方法。

```
micro api --namespace=com.example.api
```

## 例子
这里我们有一个3层架构的例子

- micro api（localhost：8080） - 作为http入口点
- api服务（go.micro.api.greeter） - 为面向公众提供服务
- 后端服务（go.micro.srv.greeter） - 内部范围服务

完整的工作示例在[这里](https://github.com/micro/examples/tree/master/greeter)

### 运行示例

先决条件：确保您正在运行服务发现，例如consul agent -dev

获取示例

```
git clone https://github.com/micro/examples
```

启动服务go.micro.srv.greeter

```
go run examples/greeter/srv/main.go
```

启动API服务go.micro.api.greeter

```
go run examples/greeter/api/api.go
```

开始 Micro API

```
micro api
```

### 查询
通过micro API进行HTTP调用

```
curl "http://localhost:8080/greeter/say/hello?name=Asim+Aslam"
```

HTTP `path/greeter/say/hello` 映射到服务 `go.micro.api.greeter` 方法`Say.Hello`

绕过api服务并通过`/rpc`直接调用后端

```
curl -d 'service=go.micro.srv.greeter' \
	-d 'method=Say.Hello' \
	-d 'request={"name": "Asim Aslam"}' \
	http://localhost:8080/rpc
```

与JSON完全相同的调用

```
$ curl -H 'Content-Type: application/json' \
	-d '{"service": "go.micro.srv.greeter", "method": "Say.Hello", "request": {"name": "Asim Aslam"}}' \
	http://localhost:8080/rpc
```

## 请求映射
Micro使用固定的命名空间和HTTP路径动态地路由到服务。

这些服务的默认命名空间是`go.micro.api`，但可以通过`--namespace`标志设置命名空间。

### 每个服务的API
我们提倡为面向公众的流量创建每个后端服务的API服务模式。这在逻辑上将服务API前端和后端服务的分开。

### RPC映射
URLs映射如下：

|Path|Service|Method|
|----|----|----|
|/foo/bar        |go.micro.api.foo     |Foo.Bar|
|/foo/bar/baz    |go.micro.api.foo     |Bar.Baz|
|/foo/bar/baz/cat|go.micro.api.foo.bar |Baz.Cat|

版本化的API URL可以很容易地映射到服务名称：

|Path|Service|Method|
|----|----|----|
|/foo/bar       |go.micro.api.foo   |Foo.Bar|
|/v1/foo/bar    |go.micro.api.v1.foo|Foo.Bar|
|/v1/foo/bar/baz|go.micro.api.v1.foo|Bar.Baz|
|/v2/foo/bar    |go.micro.api.v2.foo|Foo.Bar|
|/v2/foo/bar/baz|go.micro.api.v2.foo|Bar.Baz|

### REST映射
您可以使用API作为反向代理并使用诸如[go-restful](https://github.com/emicklei/go-restful)之类的库实现RESTful路径，从而为RESTful API提供服务。REST API服务的一个例子可以在[greeter/api/rest](https://github.com/micro/examples/tree/master/greeter/api/rest)找到。

使用`--handler=proxy`运行micro API会将代理请求反转为API名称空间内的服务。

|Path|Service|Service Path|
|----|----|----|
|/foo/bar      |go.micro.api.foo    |/foo/bar|
|/greeter      |go.micro.api.greeter|/greeter|
|/greeter/:name|go.micro.api.greeter|/greeter/:name|

使用这个处理程序意味着直接与后端服务通话，忽略任何go-micro传输插件。

## 统计仪表板

通过`--enable_stats`标志启用统计信息显示板。它将暴露在`/stats`上。

```
micro --enable_stats api
```

![](https://micro.mu/docs/images/stats.png)
