# Kubernetes部署

Micro可以在kubernetes上轻松运行，甚至可以使用kubernetes API进行服务发现。

## 配置
在[github.com/micro/kubernetes](https://github.com/micro/kubernetes)仓库为Kubernetes提供了Micro示例配置。

```
git clone https://github.com/micro/kubernetes
```

### 仓库中有什么？
目前配置为在Kubernetes上运行Micro（在Google Container Engine上测试）。

- 微型API，Web UI和Sidecar（旋转GCE负载平衡器）
- 服务 - 一些示例微服务
- run.sh - 使用kubectl启动/停止服务的简单shell脚本

## 入门
以下是我开始使用的步骤。

### 运行Kubernetes
GKE是运行托管kubernetes集群的最简单方法。什么更好？免费赠送300美元在60天期限内。

1. 让自己[免费试用](https://cloud.google.com/free-trial/)Google Container Engine
2. 访问快速[入门指南](https://cloud.google.com/container-engine/docs/quickstart)以创建群集

### 运行Micro
确保kubectl命令行在path中。

```
./run.sh start
```