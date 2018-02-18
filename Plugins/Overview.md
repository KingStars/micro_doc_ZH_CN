# 插件
Micro为所有工具提供可插拔架构。这意味着底层的实现可以被换出。

Go-Micro和Micro工具包包含不同类型的插件。从侧边栏导航以了解更多信息。

## 例子

### Go Micro
- [Etcd Registry](https://github.com/micro/go-plugins/tree/master/registry/etcd) - 使用Etcd进行服务发现
- [K8s Registry](https://github.com/micro/go-plugins/tree/master/registry/kubernetes) - 使用Kubernetes进行服务发现
- [Kafka Broker](https://github.com/micro/go-plugins/tree/master/broker/kafka) - Kafka消息总线

### 工具包
- [路由器](https://github.com/micro/go-plugins/tree/master/micro/router) - 可配置的http路由和代理
- [AWS X-Ray](https://github.com/micro/go-plugins/tree/master/micro/trace/awsxray) - 跟踪AWS X-Ray的集成
- [IP Whitelite](https://github.com/micro/go-plugins/tree/master/micro/ip_whitelist) - 白名单IP地址访问

## 知识库

开源插件可以在[github.com/micro/go-plugins](https://github.com/micro/go-plugins)上找到。
