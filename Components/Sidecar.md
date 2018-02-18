# Micro Sidecar
Micro Sidecar是用于构建高度可用和容错微服务的服务网格。

它与Netflix的[Prana](https://github.com/Netflix/Prana)，Buoyant的RPC Proxy [Linkerd](https://linkerd.io/)或Lyft的[Envoy](https://www.envoyproxy.io/)类似。

Micro Sidecar采用[go-micro](https://github.com/micro/go-micro)，具有相同的默认设置和可插拔性。

![](car.png)

可以在[examples/sidecar](https://github.com/micro/examples/tree/master/sidecar)找到许多语言的用法示例。

## API

该sidecar具有以下HTTP API。

```
- /[service]/[method]
- /broker
- /registry
- /rpc
```

## 特征
sidecar具有go-micro的所有功能。这是最相关的。

- 服务发现
- 消息总线
- RPC和代理处理程序
- 负载平衡，重试，超时
- 健康检测
- 统计界面
- 可通过go-micro插入

## 入门

### 安装

```
go get github.com/micro/micro
```

### 依赖
Sidecar使用go-micro，这意味着它有一个默认依赖关系，用于服务发现的Consul。

```
brew install consul
consul agent -dev
```

### 运行

默认情况下，在端口8081上运行Micro Sidecar。

启动Sidecar

```
micro sidecar
```

如果要在启动时自动注册应用程序，请指定应用程序服务名和地址。

```
micro sidecar --server_name=foo --server_address=127.0.0.1:9090
```

### 通过ACME使能加密
通过使用ACME提供安全服务

```
micro --enable_acme sidecar
```

可以指定一个主机白名单

```
micro --enable_acme --acme_hosts=example.com,proxy.example.com sidecar
```

### 提供TLS安全
Sidecar支持使用TLS证书安全地提供服务

```
micro --enable_tls --tls_cert_file=/path/to/cert --tls_key_file=/path/to/key sidecar
```

### 自动健康检查
用“-healthcheck_url=”启动微型边车以启用健康检查器

它执行以下操作： 
- 自动服务注册
- 定期HTTP健康检查
- 通过非200响应取消注册

```
micro sidecar --server_name=foo --server_address=127.0.0.1:9090 \
	--healthcheck_url=http://127.0.0.1:9090/health
```

## 注册

### 注册服务

```
// specify ttl as a param to expire the registration
// units ns|us|ms|s|m|h
// http://127.0.0.1:8081/registry?ttl=10s

curl -H 'Content-Type: application/json' http://127.0.0.1:8081/registry -d 
{
	"Name": "foo.bar",
	"Nodes": [{
		"Port": 9091,
		"Address": "127.0.0.1",
		"Id": "foo.bar-017da09a-734f-11e5-8136-68a86d0d36b6"
	}]
}
```

### 取消注册

```
curl -X "DELETE" -H 'Content-Type: application/json' http://127.0.0.1:8081/registry -d 
{
	"Name": "foo.bar",
	"Nodes": [{
		"Port": 9091,
		"Address": "127.0.0.1",
		"Id": "foo.bar-017da09a-734f-11e5-8136-68a86d0d36b6"
	}]
}
```

### 查询服务

```
curl http://127.0.0.1:8081/registry?service=go.micro.srv.example
{
	"name":"go.micro.srv.example",
	"nodes":[{
		"id":"go.micro.srv.example-c5718d29-da2a-11e4-be11-68a86d0d36b6",
		"address":"[::]","port":60728
	}]
}
```

## Handlers

### RPC

使用json或protobuf查询微服务。对后端的请求将使用go-micro RPC客户端进行。

使用`/[service]/[method]`

所调用服务的默认名称空间是`go.micro.srv`

```
curl -H 'Content-Type: application/json' -d '{"name": "John"}' http://127.0.0.1:8081/example/call
```

使用`/rpc`端口

```
curl -d 'service=go.micro.srv.example' \
	-d 'method=Example.Call' \
	-d 'request={"name": "John"}' http://127.0.0.1:8081/rpc
```

### Proxy

与api和web服务器一样，sidecar可以提供完整的http代理。

在命令行上启用代理处理程序。

```
micro sidecar --handler=proxy
```

URL路径中的第一个元素将与名称空间一起用作要路由到的服务。

## 请求映射
URL路径映射与Micro API相同

URL的映射如下：

|Path|Service|Method|
|---|---|---|
|/foo/bar        |go.micro.srv.foo|    Foo.Bar|
|/foo/bar/baz    |go.micro.srv.foo|    Bar.Baz|
|/foo/bar/baz/cat|go.micro.srv.foo.bar|Baz.Cat|

版本化的API URL可以很容易地映射到服务名称：

|Path|Service|Method|
|---|---|---|
|/foo/bar       |go.micro.srv.foo   |Foo.Bar|
|/v1/foo/bar    |go.micro.srv.v1.foo|Foo.Bar|
|/v1/foo/bar/baz|go.micro.srv.v1.foo|Bar.Baz|
|/v2/foo/bar    |go.micro.srv.v2.foo|Foo.Bar|
|/v2/foo/bar/baz|go.micro.srv.v2.foo|Bar.Baz|

## 事件

### 发布

```
curl -XPOST \
        -H "Timestamp: 1499951537" \
        -d "Hello World!" \
        "http://localhost:8081/broker?topic=foo"
```

### 订阅

```
conn, _, _ := websocket.DefaultDialer.Dial("ws://127.0.0.1:8081/broker?topic=foo", make(http.Header))

// optionally specify "queue=[queue name]" param to distribute traffic amongst subscribers
// websocket.DefaultDialer.Dial("ws://127.0.0.1:8081/broker?topic=foo&queue=group-1", make(http.Header))

for {
	// Read message
	_, p, err := conn.ReadMessage()
	if err != nil {
		return
	}

	// Unmarshal into broker.Message
	var msg *broker.Message
	json.Unmarshal(p, &msg)
	
	// Print message body
	fmt.Println(msg.Body)
}
```

## CLI代理
该sidecar还充当CLI访问远程环境的代理。

```
$ micro --proxy_address=127.0.0.1:8081 list services
go.micro.srv.greeter
```

## 统计仪表板
通过`--enable_stats`标志启用统计信息显示板。它将暴露在`/stats`上。

```
micro --enable_stats sidecar
```

![](stats2.png)
