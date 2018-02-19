# 工具包插件
插件是将外部代码集成到Micro工具箱的一种方式。这与Micro插件完全分开。这里使用的插件，允许您在工具包中添加额外的标志，命令和HTTP处理程序。

## 怎么运行的？
在`micro/plugin`下有一个全局插件管理器，它包含整个工具包中使用的插件。可以通过调用`plugin.Register`来注册插件。每个组件（api，web，sidecar，cli，bot）都有一个单独的插件管理器，用于注册只作为该组件一部分的添加插件。它们可以通过称为`api.Register`，`web.Register`等以相同的方式使用。

这是接口

```
// Plugin is the interface for plugins to micro. It differs from go-micro in that it's for
// the micro API, Web, Sidecar, CLI. It's a method of building middleware for the HTTP side.
type Plugin interface {
	// Global Flags
	Flags() []cli.Flag
	// Sub-commands
	Commands() []cli.Command
	// Handle is the middleware handler for HTTP requests. We pass in
	// the existing handler so it can be wrapped to create a call chain.
	Handler() Handler
	// Init called when command line args are parsed.
	// The initialised cli.Context is passed in.
	Init(*cli.Context) error
	// Name of the plugin
	String() string
}

// Manager is the plugin manager which stores plugins and allows them to be retrieved.
// This is used by all the components of micro.
type Manager interface {
        Plugins() map[string]Plugin
        Register(name string, plugin Plugin) error
}

// Handler is the plugin middleware handler which wraps an existing http.Handler passed in.
// Its the responsibility of the Handler to call the next http.Handler in the chain.
type Handler func(http.Handler) http.Handler
```

## 如何使用它
这里有一个简单的插件示例，它添加一个标志，然后打印该值。

### 插件
在顶级目录中创建一个plugin.go文件。

```
package main

import (
	"log"
	"github.com/micro/cli"
	"github.com/micro/micro/plugin"
)

func init() {
	plugin.Register(plugin.NewPlugin(
		plugin.WithName("example"),
		plugin.WithFlag(cli.StringFlag{
			Name:   "example_flag",
			Usage:  "This is an example plugin flag",
			EnvVar: "EXAMPLE_FLAG",
			Value: "avalue",
		}),
		plugin.WithInit(func(ctx *cli.Context) error {
			log.Println("Got value for example_flag", ctx.String("example_flag"))
			return nil
		}),
	))
}
```

### 构建代码
只需使用该插件构建micro。

```
go build -o micro ./main.go ./plugin.go
```

## 知识库
该工具包的插件可以在中找到[github.com/micro/go-plugins/micro](https://github.com/micro/go-plugins/tree/master/micro)。
