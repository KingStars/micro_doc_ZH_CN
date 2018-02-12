# 写一个去服务

- [写一个服务](#写一个服务)
    1. Initialisation
    2. Defining the API
    3. Generate the API interface
    4. Implement the handler
    5. Running the service
    6. The complete service
- [写一个客户端](#写一个客户端)


这是go-micro入门指南。

如果您首先喜欢更高级别的工具包概览，请查看介绍性博客文章https://micro.mu/blog/2016/03/20/micro.html。

## 写一个服务
顶级[服务接口](https://godoc.org/github.com/micro/go-micro#Service)是构建服务的主要组件。它将Go Micro的所有底层软件包整合到一个简单的界面中。

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

