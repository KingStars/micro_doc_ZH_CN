# Go Micro Plugins
Micro是一个可插拔的工具包和框架。内部功能都可以通过[go-plugins](https://github.com/micro/go-plugins)进行替换。

该工具包有一个单独的插件界面。[micro/plugin](https://github.com/micro/micro/tree/master/plugin)了解更多。

以下是关于go-micro插件的使用情况。

## 用法
可以通过以下方式将插件添加到go-micro中。通过这样做，他们可以通过命令行参数或环境变量进行设置。

### 导入插件

```
import (
	"github.com/micro/go-micro/cmd"
	_ "github.com/micro/go-plugins/broker/rabbitmq"
	_ "github.com/micro/go-plugins/registry/kubernetes"
	_ "github.com/micro/go-plugins/transport/nats"
)

func main() {
	// Parse CLI flags
	cmd.Init()
}
```

### 调用`service.Init`时也是一样

```
import (
	"github.com/micro/go-micro"
	_ "github.com/micro/go-plugins/broker/rabbitmq"
	_ "github.com/micro/go-plugins/registry/kubernetes"
	_ "github.com/micro/go-plugins/transport/nats"
)

func main() {
	service := micro.NewService(
		// Set service name
		micro.Name("my.service"),
	)

	// Parse CLI flags
	service.Init()
}
```

### 通过CLI标志使用
通过CLI标志激活

```
go run service.go --broker=rabbitmq --registry=kubernetes --transport=nats
```

### 直接使用插件
CLI标志提供了初始化插件的简单方法，但您可以自己做同样的事情。

```
import (
	"github.com/micro/go-micro"
	"github.com/micro/go-plugins/registry/kubernetes"
)

func main() {
	registry := kubernetes.NewRegistry() //a default to using env vars for master API

	service := micro.NewService(
		// Set service name
		micro.Name("my.service"),
		// Set service registry
		micro.Registry(registry),
	)
}
```

## 构建模式
您可能想要使用自动化替换插件或将插件添加到micro工具包。一个简单的方法是通过为插件导入维护一个单独的文件并在构建过程中包含它。

创建plugins.go文件

```
package main

import (
    _ “github.com/micro/go-plugins/broker/rabbitmq” 
    _ “github.com/micro/go-plugins/registry/kubernetes” 
    _ “github.com/micro/go-plugins/transport/nats” 
)

```

构建plugins.go

```
shell go build -o service main.go plugins.go
```

运行plugins

```
shell service --broker=rabbitmq --registry=kubernetes --transport=nats
```

## 用插件重建工具包
如果你想集成插件，只需将它们链接到一个单独的文件并重建。

创建一个plugins.go文件

```
import (
        // etcd v3 registry
        _ "github.com/micro/go-plugins/registry/etcdv3"
        // nats transport
        _ "github.com/micro/go-plugins/transport/nats"
        // kafka broker
        _ "github.com/micro/go-plugins/broker/kafka"
)
```

构建二进制文件

```
// For local use
go build -i -o micro ./main.go ./plugins.go

// For docker image
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags '-w' -i -o micro ./main.go ./plugins.go
```

插件标记的用法

```
micro --registry=etcdv3 --transport=nats --broker=kafka
```

## 仓库
go-micro插件可以在这里找到[github.com/micro/go-plugins](https://github.com/micro/go-plugins)。
